# 数据接口

source: `{{ page.path }}`

## IBaseDataMgr.h

主要处理交易时间, 合约信息等非交易数据

### IBaseDataMgr

基础数据管理器接口

```cpp
// 节日元组
typedef faster_hashset<uint32_t> HolidaySet;
// 节日模板结构体
typedef struct _TradingDayTpl
{
	uint32_t	_cur_tdate;
	HolidaySet	_holidays;

	_TradingDayTpl() :_cur_tdate(0){}
} TradingDayTpl;

// 数据管理器接口类
class IBaseDataMgr
{
public:
    // 获取品种信息
	virtual WTSCommodityInfo*	getCommodity(const char* exchgpid)						= 0;
	virtual WTSCommodityInfo*	getCommodity(const char* exchg, const char* pid)		= 0;
	virtual WTSCommodityInfo*	getCommodity(WTSContractInfo* ct)						= 0;

    // 获取合约信息
	virtual WTSContractInfo*	getContract(const char* code, const char* exchg = "")	= 0;
	virtual WTSArray*			getContracts(const char* exchg = "")					= 0; 

    // 获取交易时间 
	virtual WTSSessionInfo*		getSession(const char* sid)						= 0;
	virtual WTSSessionInfo*		getSessionByCode(const char* code, const char* exchg = "") = 0;
	virtual WTSArray*			getAllSessions() = 0;

    // 判断是否是节日
	virtual bool				isHoliday(const char* pid, uint32_t uDate, bool isTpl = false) = 0;

    // 计算交易日
	virtual uint32_t			calcTradingDate(const char* stdPID, uint32_t uDate, uint32_t uTime, bool isSession = false) = 0;

    // 获取分割时间
	virtual uint64_t			getBoundaryTime(const char* stdPID, uint32_t tDate, bool isSession = false, bool isStart = true) = 0;
};
```

## IDataManager.h

主要处理K线, 订单等交易数据切片

### IDataManager

交易数据管理器接口

```cpp
class IDataManager
{
public:
    // 获取各类交易数据切片
	virtual WTSTickSlice* get_tick_slice(const char* stdCode, uint32_t count, uint64_t etime = 0) { return NULL; }
	virtual WTSOrdQueSlice* get_order_queue_slice(const char* stdCode, uint32_t count, uint64_t etime = 0) { return NULL; }
	virtual WTSOrdDtlSlice* get_order_detail_slice(const char* stdCode, uint32_t count, uint64_t etime = 0) { return NULL; }
	virtual WTSTransSlice* get_transaction_slice(const char* stdCode, uint32_t count, uint64_t etime = 0) { return NULL; }
	virtual WTSKlineSlice* get_kline_slice(const char* stdCode, WTSKlinePeriod period, uint32_t times, uint32_t count, uint64_t etime = 0) { return NULL; }

    // 获取最新tick数据
	virtual WTSTickData* grab_last_tick(const char* code) { return NULL; }
};
```

## IDataFactory.h

### IDataFactory

数据工厂接口, 主要用于各类数据拼接

```cpp
class IDataFactory
{
public:
	// 利用tick数据更新K线, klineData: K线数据, tick: tick数据, sInfo: 交易时间模板
	virtual WTSBarStruct*	updateKlineData(WTSKlineData* klineData, WTSTickData* tick, WTSSessionInfo* sInfo)						= 0;

	// 利用基础周期K线数据更新K线, klineData: K线数据, newBasicBar: 基础周期K线数据, sInfo: 交易时间模板
	virtual WTSBarStruct*	updateKlineData(WTSKlineData* klineData, WTSBarStruct* newBasicBar, WTSSessionInfo* sInfo)				= 0;

	// 从K线切片提取K线, baseKline: 基础周期K线, period: 基础周期(m1/m5/day), times: 周期倍数, sInfo: 交易时间模板, bIncludeOpen: 是否包含未闭合的K线
	virtual WTSKlineData*	extractKlineData(WTSKlineSlice* baseKline, WTSKlinePeriod period, uint32_t times, WTSSessionInfo* sInfo, bool bIncludeOpen = true) = 0;

	// 从tick切片提取K线, ayTicks: tick数据, seconds: 目标周期, sInfo: 交易时间模板, bUnixTime: tick时间戳是否是unixtime
	virtual WTSKlineData*	extractKlineData(WTSTickSlice* ayTicks, uint32_t seconds, WTSSessionInfo* sInfo, bool bUnixTime = false) = 0;

	// 合并K线, klineData: 目标K线, newKline: 待合并的K线
	virtual bool			mergeKlineData(WTSKlineData* klineData, WTSKlineData* newKline)											= 0;
};
```

## IDataReader.h

### IDataReaderSink

数据读取模块回调接口, 主要用于数据读取模块向调用模块回调

```cpp
class IDataReaderSink
{
public:
	// K线闭合事件回调, stdCode: 标准品种代码(如SSE.600000,SHFE.au.2005), period: K线周期, newBar: 闭合的K线结构指针
	virtual void on_bar(const char* stdCode, WTSKlinePeriod period, WTSBarStruct* newBar) = 0;

	// 所有缓存的K线全部更新的事件回调, updateTime: K线更新时间(精确到分钟, 如202004101500)
	virtual void on_all_bar_updated(uint32_t updateTime) = 0;

	// 获取基础数据管理接口指针
	virtual IBaseDataMgr*	get_basedata_mgr() = 0;

	// 获取主力切换规则管理接口指针
	virtual IHotMgr*		get_hot_mgr() = 0;

	// 获取当前日期,格式(20100410)
	virtual uint32_t	get_date() = 0;

	// 获取当前1分钟线的时间(这里的分钟线时间是处理过的1分钟线时间,如现在是9:00:32秒,真实事件为0900,但是对应的1分钟线时间为0901)
	virtual uint32_t	get_min_time() = 0;

	// 获取当前的秒数,精确到毫秒,如37,500
	virtual uint32_t	get_secs() = 0;

	// 输出数据读取模块的日志
	virtual void		reader_log(WTSLogLevel ll, const char* fmt, ...) = 0;
};

// 历史数据加载器的回调函数, obj: 回传用(原样返回即可), bars: K线数据, count: K线条数
typedef void(*FuncReadBars)(void* obj, WTSBarStruct* bars, uint32_t count);

// 加载复权因子回调, obj: 回传用(原样返回即可), stdCode: 合约代码, dates: 日期, factors: 复权因子, (如果是后复权, 则factor不为1.0; 如果是前复权, 则factor必须为1.0)
typedef void(*FuncReadFactors)(void* obj, const char* stdCode, uint32_t* dates, double* factors, uint32_t count);
```


### IHisDataLoader

历史数据加载器

```cpp
class IHisDataLoader
{
public:
    // loadFinalHisBars: 系统认为是最终所需数据, 不再进行加工, 例如复权数据, 主力合约数据; loadRawHisBars: 是加载未加工的原始数据的接口

	// 加载最终历史K线数据, obj: 回传用(原样返回即可), stdCode: 合约代码, period: K线周期, cb: 回调函数
	virtual bool loadFinalHisBars(void* obj, const char* stdCode, WTSKlinePeriod period, FuncReadBars cb) = 0;

	// 加载原始历史K线数据, obj: 回传用(原样返回即可), stdCode: 合约代码, period: K线周期, cb: 回调函数
	virtual bool loadRawHisBars(void* obj, const char* stdCode, WTSKlinePeriod period, FuncReadBars cb) = 0;

	// 加载全部除权因子
	virtual bool loadAllAdjFactors(void* obj, FuncReadFactors cb) = 0;

	// 加指定合约除权因子
	virtual bool loadAdjFactors(void* obj, const char* stdCode, FuncReadFactors cb) = 0;
};
```

### IDataReader

数据读取接口, 向核心模块提供行情数据(tick、K线)读取接口

```cpp
class IDataReader
{
public:
	IDataReader(){}
	virtual ~IDataReader(){}

public:
	// 初始化数据读取模块, cfg: 模块配置项, sink: 模块回调接口
	virtual void init(WTSVariant* cfg, IDataReaderSink* sink, IHisDataLoader* loader = NULL) { _sink = sink; _loader = loader; }

	// 分钟线闭合事件处理接口, uDate: 闭合的分钟线日期(如20200410, 不是交易日), uTime: 闭合的分钟线的分钟时间(如1115), endTDate: 如果闭合的分钟线是交易日最后一条分钟线, 则endTDate为当前交易日(如20200410), 其他情况为0
	virtual void onMinuteEnd(uint32_t uDate, uint32_t uTime, uint32_t endTDate = 0) = 0;

	// 读取tick数据切片, stdCode: 标准品种代码(如SSE.600000, SHFE.au.2005), count: 要读取的tick条数, etime: 结束时间(精确到毫秒, 格式如yyyyMMddhhmmssmmm, 如果要读取到最后一条, etime为0, 默认为0)
	virtual WTSTickSlice*	readTickSlice(const char* stdCode, uint32_t count, uint64_t etime = 0) = 0;

	// 读取逐笔委托数据切片
	virtual WTSOrdDtlSlice*	readOrdDtlSlice(const char* stdCode, uint32_t count, uint64_t etime = 0) = 0;

	// 读取委托队列数据切片
	virtual WTSOrdQueSlice*	readOrdQueSlice(const char* stdCode, uint32_t count, uint64_t etime = 0) = 0;

	// 读取逐笔成交数据切片
	virtual WTSTransSlice*	readTransSlice(const char* stdCode, uint32_t count, uint64_t etime = 0) = 0;

	// 读取K线序列, 并返回一个存储容器类, stdCode: 标准品种代码(如SSE.600000, SHFE.au.2005), period: K线周期, count: 要读取的K线条数, etime: 结束时间(格式yyyyMMddhhmm)
	virtual WTSKlineSlice*	readKlineSlice(const char* stdCode, WTSKlinePeriod period, uint32_t count, uint64_t etime = 0) = 0;

	// 获取个股指定日期的复权因子, stdCode: 标准品种代码(如SSE.600000), date: 指定日期(格式yyyyMMdd, 默认为0, 为0则按当前日期处理)
	virtual double		getAdjFactorByDate(const char* stdCode, uint32_t date = 0) { return 1.0; }

protected:
	IDataReaderSink*	_sink;
	IHisDataLoader*		_loader;
};

//创建数据存储对象
typedef IDataReader* (*FuncCreateDataReader)();
//删除数据存储对象
typedef void(*FuncDeleteDataReader)(IDataReader* store);
```

## IDataWriter.h

数据落地接口

### IDataWriterSink

数据写入回调接口

```cpp
class IDataWriterSink
{
public:
	
	virtual IBaseDataMgr* getBDMgr() = 0;

	virtual bool canSessionReceive(const char* sid) = 0;

	virtual void broadcastTick(WTSTickData* curTick) = 0;

	virtual void broadcastOrdQue(WTSOrdQueData* curOrdQue) = 0;

	virtual void broadcastOrdDtl(WTSOrdDtlData* curOrdDtl) = 0;

	virtual void broadcastTrans(WTSTransData* curTrans) = 0;

	virtual CodeSet* getSessionComms(const char* sid) = 0;
    // 获取交易日期
	virtual uint32_t getTradingDate(const char* pid) = 0;

	// 处理解析模块的日志, ll: 日志级别, message: 日志内容
	virtual void outputWriterLog(WTSLogLevel ll, const char* format, ...) = 0;
};
```

### IHisDataDumper

```cpp
class IHisDataDumper
{
public:
    // 获取历史数据接口
	virtual bool dumpHisBars(const char* stdCode, const char* period, WTSBarStruct* bars, uint32_t count) = 0;
	virtual bool dumpHisTicks(const char* stdCode, uint32_t uDate, WTSTickStruct* ticks, uint32_t count) = 0;
	virtual bool dumpHisOrdQue(const char* stdCode, uint32_t uDate, WTSOrdQueStruct* items, uint32_t count) { return false; }
	virtual bool dumpHisOrdDtl(const char* stdCode, uint32_t uDate, WTSOrdDtlStruct* items, uint32_t count) { return false; }
	virtual bool dumpHisTrans(const char* stdCode, uint32_t uDate, WTSTransStruct* items, uint32_t count) { return false; }
};

typedef faster_hashmap<std::string, IHisDataDumper*> ExtDumpers;
```

### IDataWriter

数据落地接口

```cpp
class IDataWriter
{
public:
	IDataWriter():_sink(NULL){}

	virtual bool init(WTSVariant* params, IDataWriterSink* sink) { _sink = sink; return true; }

	virtual void release() = 0;

	void	add_ext_dumper(const char* id, IHisDataDumper* dumper) { _dumpers[id] = dumper; }

public:
	// 写入相关数据
	virtual bool writeTick(WTSTickData* curTick, bool bNeedSlice = true) = 0;
	virtual bool writeOrderQueue(WTSOrdQueData* curOrdQue) = 0;
	virtual bool writeOrderDetail(WTSOrdDtlData* curOrdDetail) = 0;
	virtual bool writeTransaction(WTSTransData* curTrans) = 0;

	virtual void transHisData(const char* sid) = 0;

	virtual bool isSessionProceeded(const char* sid) = 0;

	virtual WTSTickData* getCurTick(const char* code, const char* exchg = "") = 0;

protected:
	ExtDumpers			_dumpers;
	IDataWriterSink*	_sink;
};
```