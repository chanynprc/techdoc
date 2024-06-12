## AWS Zero-ETL

### AWS Zero-ETL概述

AWS Zero-ETL并非不需要ETL，还是需要ETL过程的，只是让这个过程更加简单方便。AWS Zero-ETL更多关注在E和L上，当前对T的支持有限，T更多地需要放到L之后（即ELT），在Redshift中处理。AWS Zero-ETL有助于构建生态，提供一体化能力。

AWS Zero-ETL主要由几种数据集成和处理方式组合而成：

- Amazon Aurora Zero ETL to Amazon Redshift：关系型数据库通过CDC复制技术，把全量、增量数据复制到Redshift
- Amazon Redshift Copy Job：S3增量文件自动写入Redshift
- Amazon Redshift streaming ingestion：数据实时写入Redshift，结合物化视图做T
- 联邦查询：Redshift直接访问外部数据源
- AI集成：应用数据写入Redshift或S3，SageMaker和Redshift做T

### AWS Zero-ETL对于客户业务的影响

在业务人员方面：

- 在没有Zero-ETL时，客户的人员配置包含业务人员/业务数据分析师，以及数据开发工程师。业务开发需要业务人员/业务数据分析师提出业务需求（包含数据源、计算逻辑等），然后由数据开发工程师根据业务理解，进行数据采集、任务编排等，再由业务人员/业务数据分析师进行验收，可能要重复几轮。

- 有Zero-ETL后，数据已经通过Zero-ETL被集成到数据仓库，业务人员/业务数据分析师可直接通过SQL进行查询分析，数据开发工程师可以从繁琐的ETL过程中解放，专注数据分析或者向业务人员/业务数据分析师转型。

Zero-ETL为客户带来如下业务价值：

- 简化数据管道的构建和维护工作
- 打破组织中的数据孤岛
- 可以多源汇聚，有助于数据整合
- 可以对事务数据进行近乎实时的分析和机器学习
- 方便地通过Redshift增强数据分析
