# 简介

数据仓库 ，由数据仓库之父W.H.Inmon于1990年提出。

数据仓库是企业数据的大脑，是企业无形的资产。

通过合理规划，科学建模方法论，建立面向主题，时变高效稳定集成的数据中台，方便决策，提升组织运行效率。

## 数据中台

![严选数据中台](https://oss.dataown.cn/data/2020/9/6cc07a4853ddbbf9.webp)

## 模型层次
[Linux的创始人Linus](https://zh.wikipedia.org/wiki/%E6%9E%97%E7%BA%B3%E6%96%AF%C2%B7%E6%89%98%E7%93%A6%E5%85%B9)有一段关于“什么才是优秀程序员”的话：`“烂程序员关心的是代码，好程序员关心的是数据结构和它们之间的关系”`，最能够说明数据模型的重要性。

* 性能
    * 每一个数据分层都有它的作用域，使用表的时候能更方便地定位和理解，减少数据的 I/O 吞吐。
* 成本
    * 良好的数据模型能极大地减少不必要的数据冗余，也能实现计算结果复用，极大的降低大数据系统中的存储和计算成本
* 效率
    * 规范数据分层，通用的中间层数据，能够减少极大的重复计算，提高获取数据的效率
* 质量
    * 复杂的任务分解成多个步骤，每一层只处理单一的步骤，当出现问题时修复快，业务变动只需要在基础层处理兼容
* 高内聚低耦合
    * 主题之内或各个完整意义的系统内数据的高内聚
    * 主题之间或各个完整意义的系统间数据的松耦合


### 模型分层

<table>
    <tr>
      <th>模型层次</th>
      <th>英文名</th>
      <th>中文名</th>
      <th>层次定义</th>
    <tr>
      <td align="center"><a href="model_level.md/#app">APP</a></td>
      <td align="center">Application</a></td>
      <td align="center">应用数据层</a></td>
      <td align="center">该层级的主要功能是提供差异化的数据服务（数据服务接口）、满足业务方的需求</a></td>
    </tr>
    <tr>
      <td align="center"><a href="model_level.md/#rpt">RPT</a></td>
      <td align="center">Report</a></td>
      <td align="center">报表数据层</a></td>
      <td align="center">该层级的主要功能是提供报表数据，在该层级实现报表（数易、邮件报表）、自助取数等需求。</a></td>
    </tr>
    <tr>
      <td align="center"><a href="model_level.md/#dm">DM</a></td>
      <td align="center">Data Warehouse Summary</a></td>
      <td align="center">数据集市层</a></td>
      <td align="center">该层次主要功能是加工多维度冗余的宽表（解决复杂的查询）、多角度分析的汇总表。</a></td>
    </tr>
    <tr>
      <td align="center"><a href="model_level.md/#dws">DWS</a></td>
      <td align="center">Data Warehouse Summary</a></td>
      <td align="center">汇总数据层</a></td>
      <td align="center">基于轻度汇总层，结合数据的具体应用场景和时间周期，沉淀出常用的派生指标。统一计算口径，避免重复计算，可复用</a></td>
    </tr>
    <tr>
      <td align="center"><a href="model_level.md/#dwb">DWB</a></td>
      <td align="center">Data Warehouse Base</a></td>
      <td align="center">基础数据层</a></td>
      <td align="center">面向应用主题的、统一的轻度汇总层，所有的基础数据、业务规则和业务实体的基础指标库以及多维模型都在这里统一计算口径、统一建模，大量基础指标库以及多维模型在该层实现。该层级以应用需求为驱动进行设计，实现不跨数据域的轻度汇总计算。</a></td>
    </tr>
    <tr>
      <td align="center"><a href="model_level.md/#dwd">DWD</a></td>
      <td align="center">Data Warehouse Detail</a></td>
      <td align="center">明细数据层</a></td>
      <td align="center">该层的主要功能是基于数据域的划分，抽象出各个数据域的业务过程，以业务的数据化还原为驱动设计模型，完成数据整合，提供统一的明细数据来源。在该层级完成数据的清洗、重定义、整合分类功能。主要采用基于业务过程建模的事务事实表以及快照事实表设计。</a></td>
    </tr>
    <tr>
      <td align="center"><a href="model_level.md/#dim">DIM</a></td>
      <td align="center">Dimension</a></td>
      <td align="center">维度层</a></td>
      <td align="center">该层主要存储简单、静态、代码类的维表，包括从OLTP层抽取转换维表、根据业务或分析需求构建的维表以及仓库技术维表如日期维表等</a></td>
    </tr>
    <tr>
      <td align="center"><a href="model_level.md/#ods">ODS</a></td>
      <td align="center">Operational Data Store</a></td>
      <td align="center">操作数据层</a></td>
      <td align="center">该层级主要功能是存储从源系统直接获得的数据（数据从数据结构、数据之间的逻辑关系上都与源系统基本保持一致）。实现某些业务系统字段的数据仓库技术处理、少量的基础的数据清洗（比如脏数据过滤、字符集转换、维值处理）、生成增量数据表。</a></td>
    </tr>
    <tr>
      <td align="center"><a href="model_level.md/#rods">RODS</a></td>
      <td align="center">Realtime Operational Data Store</a></td>
      <td align="center">准实时操作数据层</a></td>
      <td align="center">主要是日志数据，通过flume、flink等工具生成按时间、大小的滚动日志</a></td>
    </tr>
    <tr>
      <td align="center"><a href="model_level.md/#stage">STAGE</a></td>
      <td align="center">Stage</a></td>
      <td align="center">数据接入层</a></td>
      <td align="center">数仓分全量抽取加载和增量抽取加载，少量的基础的数据清洗（比如脏数据过滤、字符集转换、维值处理，'\n'、'\t' 等处理），对敏感数据进行加密处理，按数据来源进行命名加以区分</a></td>
    </tr>
  </table>
<br/>

### 模型分层

![](https://oss.dataown.cn/data/2020/8/691e7dc1b029437e246b8ac2dc6fd754.png)

### 功能架构

![](https://oss.dataown.cn/data/2020/8/42b6e813f619c338.png)








