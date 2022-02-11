# 交易时间接口

source: `{{ page.path }}`

## ISessionMag.h

### ISessionMgr

时间模板管理器接口

```cpp
class ISessionMgr
{
public:
	// 获取合约所属的时间模板对象指针, code: 合约代码, exchg: 交易所代码
	virtual WTSSessionInfo* getSession(const char* code, const char* exchg = "")	= 0;
};
```