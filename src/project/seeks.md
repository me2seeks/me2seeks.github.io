# seek
## 设计数据库表

爬虫的数据库特别简单,一个表即可。这个表里面存着页面的 URL 和爬来的标题以及网页文字内容。

```sql
CREATE TABLE `pages` (
  `id` int unsigned NOT NULL AUTO_INCREMENT,
  `url` varchar(768) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci DEFAULT NULL COMMENT '网页链接',
  `host` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci DEFAULT NULL COMMENT '域名',
  `dic_done` tinyint DEFAULT '0' COMMENT '已拆分进词典',
  `craw_done` tinyint NOT NULL DEFAULT '0' COMMENT '已爬',
  `craw_time` timestamp NOT NULL DEFAULT '2001-01-01 00:00:00' COMMENT '爬取时刻',
  `origin_title` varchar(2000) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci DEFAULT NULL COMMENT '上级页面超链接文字',
  `referrer_id` int NOT NULL DEFAULT '0' COMMENT '上级页面ID',
  `scheme` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci DEFAULT NULL COMMENT 'http/https',
  `domain1` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci DEFAULT NULL COMMENT '一级域名后缀',
  `domain2` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci DEFAULT NULL COMMENT '二级域名后缀',
  `path` varchar(2000) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci DEFAULT NULL COMMENT 'URL 路径',
  `query` varchar(2000) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci DEFAULT NULL COMMENT 'URL 查询参数',
  `title` varchar(1000) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci DEFAULT NULL COMMENT '页面标题',
  `text` longtext CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci COMMENT '页面文字',
  `created_at` timestamp NOT NULL DEFAULT '2001-01-01 08:00:00' COMMENT '插入时间',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
```

爬虫有一个极好的特性：自我增殖。每一个网页里,基本都带有其他网页的链接,这样我们就可以道生一,一生二,二生三,三生万物了。

此时,我们只需要找一个导航网站,手动把该网站的链接插入到数据库里,爬虫就可以开始运作了。

使用`spf13/cobra`和`spf13/viper`分别作为CLI和配置文件读取.


  1. 循环展开 待爬列表
  2. 针对每一个 URL,使用 curl 工具类获取网页文本
  3. 解析网页文本,提取出标题和页面中含有的超链接
  4. 将标题、一级域名后缀、URL 路径、插入时间等信息补充完全,更新到这一行数据上
  5. 将页面上的超链接插入 pages 表,我们的网页库第一次扩充了!


## 优化

### 重用HTTP客户端防止内存泄漏

GO中启动一个协程的内存开销相对较小。初始化时,每个`goroutine`会被分配一个较小的栈空间,默认情况下通常是2KB大小。但在并发数十万协程的时候。每个协程的内存消耗积累起来也是巨大的,容易OOM。

所以选择[全局 client](https://req.cool/zh/docs/tutorial/best-practices/#%e9%87%8d%e7%94%a8-client)来重用HTTP

```go
package tools
import ... //省略

var (
    // 全局重用 client 对象,4 秒超时,不跟随 301 302 跳转
	client = req.C().SetTimeout(time.Second * 4).SetRedirectPolicy(req.NoRedirectPolicy())
	logger = slog.Default().With("Curl")
)
// 返回 document 对象和状态码
func Curl(status models.Status, ch chan int) (*goquery.Document, int) {}
```
爬流程

```go
var statusArr []models.Status
// 使用redis list结构的rpop 获取待爬取列表	
for i := 0; i < 256*maxNumber; i++ {
		jsonString := db.Rdb.RPop(db.Ctx, "need_craw_list").Val()

		var _status models.Status
		err := json.Unmarshal([]byte(jsonString), &_status)
		if err != nil {
			continue
		}
		statusArr = append(statusArr, _status)
	}
if len(statusArr) == 0 {
		//睡眠一分钟后再爬
	}
	// 创建 channel 数组
	chs := make([]chan int, validCount)
	for k, v := range statusArr {
		chs[k] = make(chan int)
        // 开爬
		go craw(v, chs[k], k)
	}

	results := make(map[int]int)
	for _, ch := range chs {
		r := <-ch
		_, prs := results[r]
		if prs {
			results[r]++
		} else {
			results[r] = 1
		}
	}

```

craw

```go
func craw(status models.Page, ch chan int, index int) {
  // 调用 CURL 工具类爬到网页
  doc, chVal := tools.Curl(status, ch)

  // 对 doc 的处理

  // 最重要的一步,向 chennel 发送 状态码,该动作是协程结束的标志
  ch <- chVal
  return
}
```

### `MySql`优化

现在`MySql`成为整个流程最慢的一环,`pages`表一天就要增加几百万行,innodb的B=数树高度很快到达三层(高度为3的B+树大概可以存放1170 × 1170 × 16 = 21902400 )。这时就需要对`MySql`做性能优化。

### 索引

收益最大的是加索引,在磁盘容量够用的情况下,索引可以起到几倍到几个数量级的性能提升。给URL添加索引,因为我们每爬一个URL都要查询一下它是否已经在表中,这个动作的频率是非常高的。

### 分库分表

不同的URL之间没有多少关系,所以与分库分表非常契合。根据URL将数据均匀的分散开来。

在对每一个URL进行MD5后取前两位,分出 16*16 = 256 个表

1. 计算URL的MD5 散列值
2. 取前两位十六进制数
3. 拼接成类似的 `pages_of` 的表名

```go
tableName := table + "_" + tools.GetMD5Hash(url)[0:2]
```

### 拆分`page`和`status`

pages表中存在16个字段,但在爬取的过程中只用的到五个字段 `id` `url` `host` `craw_done` `craw_time` 。pages表中存放 `longtext`的text字段存放的html页面。这时一个页(16k)可能存不下一条记录,这个时候就会发生行溢出。多的数据会存放到另外的`溢出页`。 `buffer pool `会频繁失效。

为了爬的更快,我为`pages_0f`表打造了只包含上面五个字段的`status_0f`兄弟表,数据从 pages 表里面复制而来,承担一些频繁读写任务：

1. 检查 URL 是否已经在库,即如果以前别的页面上已经出现了这个 URL 了,本次就不需要再入库了
2. 找出下一批需要爬的页面,即`craw_done=0`的 URL
3. craw_time 承担日志的作用,用于统计过去一段时间的爬虫效率

除了这些高频操作,存储页面 HTML 和标题等信息的低频操作是可以直接入`paqes_0f`仓库表的。

### 实时读取URL改为后台定时读取

随着单表数据量的逐渐提升,每一轮开始时从数据库里面批量读出需要爬的 URL 成了一个相对耗时的操作,即便每张表只需要 500ms,那轮询 256 张表总耗时也达到了 128 秒之多,这是无法接受的。还不能同时读取256张表。因为`MySql`的连接数的宝贵的。

```go
// 在 main() 中注册定时任务
c := cron.New(cron.WithSeconds())
// 每 20 秒执行一次 prepareStatusesBackground 函数
c.AddFunc("*/20 * * * * *", prepareStatusesBackground)
go c.Start()

// prepareStatusesBackground 函数中,使用 LPush 向有序列表的头部插入 URL
for _, v := range _statusArray {
  taskBytes, _ := json.Marshal(v)
  db.Rdb.LPush(db.Ctx, "need_craw_list", taskBytes)
}

// 每一轮都使用 RPop 从有序列表的尾部读取需要爬的 URL
var statusArr []models.Status
maxNumber := 1 // 放大倍数,控制每一批的 URL 数量
for i := 0; i < 256*maxNumber; i++ {
  jsonString := db.Rdb.RPop(db.Ctx, "need_craw_list").Val()
  var _status models.Status
  err := json.Unmarshal([]byte(jsonString), &_status)
  if err != nil {
    continue
  }
  statusArr = append(statusArr, _status)
}
```

### 控制单个域名爬取的频率

`sync.map` 是内存安全的,它来做为计数器会遇到以下问题

- 它适用于读多写少的场景,而我们的记录单个域名的爬取次数需要更新,需要频繁更新计数器,导致性能问题。

- 缺乏过期策略

所以选择`Redis` 记录某条URL是否存储。

```go
// 我们使用一个 Hash 来存储 URL 是否存在的状态
statusHashMapKey := "spider_status_exist"
statusExist := db.Rdb.HExists(db.Ctx, statusHashMapKey, _url).Val()
// 若 HashMap 中不存在,则查询或插入数据库
if !statusExist {
  // 不存在则创建这行 page,存在则更新信息
  // 无论是否新插入了数据,都将 _url 入 HashMap
  db.Rdb.HSet(db.Ctx, statusHashMapKey, _url, 1).Err()
}
```

唯一问题是不能运行太长,`spider_status_exist` 会占用大量内存。

统计信息也可以使用`Redis`来记录

```go
// 过去一分钟爬到了多少个页面的 HTML
allStatusKey := "spider_all_status_in_minute_" + strconv.Itoa(int(time.Now().Unix())/60)
// 计数器加 1
db.Rdb.IncrBy(db.Ctx, allStatusKey, 1).Err()
// 续命 1 小时
db.Rdb.Expire(db.Ctx, allStatusKey, time.Hour).Err()

// 过去一分钟从新爬到的 HTML 里面提取出了多少个新的待爬 URL
newStatusKey := "spider_new_status_in_minute_" + strconv.Itoa(int(time.Now().Unix())/60)
// 计数器加 1
db.Rdb.IncrBy(db.Ctx, newStatusKey, 1).Err()
// 续命 1 小时
db.Rdb.Expire(db.Ctx, newStatusKey, time.Hour).Err()
```

### 抑制暴涨的数据库连接数

在协程使用之下,我们可以轻易写出超高并行的代码,把 CPU 全部吃完,但是,并行的协程多了以后,数据库的连接数压力也开始暴增。MySQL 默认的最大连接数只有 151,根据我的实际体验,哪怕是一个协程一个连接,我们这个爬虫也可以轻易把连接数干到数万。

除了协程之外,分库分表对连接数的的暴增也负有不可推卸的责任。为了提升单条 SQL 的性能,我们给单台数据库服务器分了 256 张表,这种情况下,以前的一个连接+一条 SQL 的状态会突然增加到 256 个连接和 256 条 SQL,如果我们不加以限制的话,可以说协程+分表一启动,你就一定会收到海量的`Too many connections`报错。

这时需要设置 `Gorm`的 “单线程最大连接数”

```go
dbdb0, _ := _db0.DB()
dbdb0.SetMaxIdleConns(1)
dbdb0.SetMaxOpenConns(100)
dbdb0.SetConnMaxLifetime(time.Hour)
```

![1](https://github.com/me2seeks/seek/blob/main/images/1.png)

## 使用倒排索引生成字典

### 解释倒排索引

1. 我们有一个表 titles,含有两个字段,ID 和 text,假设这个表有 100 行数据,其中第一行 text 为“爬虫工作流程”,第二行为“制造真正的生产级爬虫”
2. 我们对这两行文本进行分词,第一行可以得到“爬虫”、“工作”、“流程”三个词,第二行可以得到“制造”、“真正的”、“生产级”、“爬虫”四个词
3. 我们把顺序颠倒过来,以词为 key,以①`titles.id` ②`,` ③`这个词在 text 中的位置` 这三个元素拼接在一起为一个`值`,不同 text 生成的`值`之间以 - 作为间隔,对数据进行“反向索引”,可以得到：
   1. 爬虫: 1,0-2,8
   2. 工作：1,2
   3. 流程：1,4
   4. 制造：2,0
   5. 真正的：2,2
   6. 生产级：2,5

### 生成倒排索引数据

我使用`yanyiwu/gojieba`这个库来调用结巴分词,按照以下步骤对我爬到的每一个 HTML 文本进行分词并归类：

1. 分词,然后循环处理这些词：
2. 统计词频：这个词在该 HTML 中出现的次数
3. 记录下这个词在该 HTML 中每一次出现的位置,从 0 开始算
4. 计算该 HTML 的总长度,搜索算法需要
5. 按照一定格式,组装成倒排索引值,形式如下：

```go
// 分表的顺序,例如 0f 转为十进制为 15
strconv.Itoa(i) + "," +
// pages.id 该 URL 的主键 ID
strconv.Itoa(int(pages.ID)) + "," +
// 词频：这个词在该 HTML 中出现的次数
strconv.Itoa(v.count) + "," +
// 该 HTML 的总长度,BM25 算法需要
strconv.Itoa(textLength) + "," +
// 这个词出现的每一个位置,用逗号隔开,可能有多个
strings.Join(v.positions, ",") +
// 不同 page 之间的间隔符
"-"
```

使用`Redis`作为词典数据的中转站,在 `Redis` 中针对每一个词生成一个 List,把倒排出来的索引插入到尾部：

```go
db.Rdb10.RPush(db.Ctx, word, appendSrting)
```

### 使用定时任务将`Redis`词典数据搬运到`MySql`

词典的生成是计算密集型工作,所以使用定时任务来异步执行

1. 随机获取一个 key
2. 判断该 key 的长度,只有大于等于 2 的进入下一步
3. 把最后一个索引值留下,前面的元素一个一个`LPop`（弹出头部）出来,拼接在一起
4. 汇集一批 2000 个随机词的结果,append 到数据库该词现有索引值的后面

有协程和`Redis`的使用后,分词加倒排索引的速度快起来了,但是如果选择一个词一个词地 append 值,会发现 MySQL 又双叒叕变的超慢,又要优化 MySQL 了

### 使用`worker`模式

如果获取一个redis中的字典开一个`goroutine`的话,就难以控制暴涨的`goroutine`数,所以选择`worker`模式

```go
	// 一步转移的字典条数
	oneStep := 1000
	// 定义worker 
	worker := func(tasksChan <-chan int, resultsChan chan<- *WordAndAppendSrting, wg *sync.WaitGroup) {
		defer wg.Done()
		for range tasksChan {
			// 进行实际的工作,并将结果发送到结果channel
			resultsChan <- asyncGetWordAndAppendSrting()
		}
	}

	tasksChan := make(chan int)
	resultsChan := make(chan *WordAndAppendSrting, 200)
	var wg sync.WaitGroup

	for i := 0; i < 10; i++ {
		wg.Add(1)
		go worker(tasksChan, resultsChan, &wg)
	}

	go func() {
		for result := range resultsChan {
			if result.word != "" {
				_, prs := needUpdate[result.word]
				if prs {
					needUpdate[result.word] += result.appendString
				} else {
					needUpdate[result.word] = result.appendString
				}
			}
		}
	}()
	// 发布任务
	for i := 0; i < oneStep; i++ {
		tasksChan <- i
	}
	// 发送完毕后关闭任务channel
	close(tasksChan) 
	// 等待所有工人完成任务
	wg.Wait()
	// 工人已经完成,关闭结果channel
	close(resultsChan)
```

### 事务的妙用：MySQL 高速批量插入

一次需要更新数据库多条词典,这时候就需要一次性 update多条数据

```go
tx.Exec(`START TRANSACTION`)

// 需要批量执行的 update 语句
for w, s := range needUpdate {
  tx.Exec(`UPDATE word_dics SET positions = concat(ifnull(positions,''), ?) where name = ?`, s, w)
}

tx.Exec(`COMMIT`)
```
