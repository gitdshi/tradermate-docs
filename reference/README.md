# Reference 参考资料

此目录包含 TraderMate 项目引用的外部数据源、API和第三方服务的技术参考文档。

## 内容

| 文档 | 描述 | 用途 |
|------|------|------|
| **[tushare_stock_endpoints.md](./tushare_stock_endpoints.md)** | Tushare股票API端点与数据库表映射 | 数据同步开发参考 |
| **（待补充）** | 其他外部API或数据源 | 未来扩展 |

---

## 外部资源链接

### 数据源

- **Tushare Pro**: https://tushare.pro/
  - 股票基础数据、日线行情、财务数据
  - 需要申请 token (免费额度有限)
  - 完整端点列表: `tushare_stock_endpoints.md`

- **AkShare**: https://akshare.readthedocs.io/
  -  alternative 数据源 (开源免费)
  - 用于备份或补充数据

### 交易平台

- **VeighNa (原 vn.py)**: https://www.veighna.com/
  - 量化交易框架
  - TraderMate 集成的策略回测引擎

### 技术栈

- **FastAPI**: https://fastapi.tiangolo.com/
- **React**: https://react.dev/
- **Tailwind CSS**: https://tailwindcss.com/
- **Redis**: https://redis.io/
- **MySQL**: https://www.mysql.com/

---

## 使用说明

这些参考资料用于开发人员理解外部数据源的结构和 TraderMate 如何映射到内部数据库。

**需要更新时**:
- 如果 Tushare API 有新增或变更，更新 `tushare_stock_endpoints.md`
- 新增数据源时，创建类似格式的参考文档

---

**维护者**: @coder, @operator  
**最后更新**: 2026-03-03