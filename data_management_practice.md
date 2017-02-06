# 数据处理练习

除了练习数据处理外，本次是针对在Finance领域十分重要的2项methodology：构造投资组合和事件研究。

Finance研究股票收益从来不是以个股为单位，而是以投资组合为单位，所以投资组合的建造，平均收益的求取就比较重要。这里选择最经典的Fama French 3 factor，其他投资组合也是用类似的方法建立。

事件研究是Finance另一重要的methodology，这里参考Dellavigna and Pollet (2009)对盈余公告效应的研究，用中国数据复制其主要结果。

需要的数据来自 __CSMAR__ :

* 股票市场系列-股票市场交易
    + 个股交易数据
        + 月个股回报率文件TRD_Mnth
        + 日个股回报率文件TRD_Dalyr
    + 汇率及利率
        + 无风险利率文件TRD_Nrrate
    + 指数信息
        + 指数文件Trd_Index
    + Fama-French因子
        + 三因子模型指标（日）Stk_mkt_thrfacday

* 公司研究系列
    + 财务报表
        + 资产负债表FS_Combas
    + 年中季报公告日期
        + 年中季报基本情况文件IAR_Rept
    + 分析师预测
        + 分析师预测指标文件AF_Forecast
        + 实际对比情况-实际指标文件AF_Actual


## 1.	投资组合构造

(a) 行过滤操作。对TRD_Mnth剔除B股数据，对FS_Combas只保留年末12月31日的合并财报的数据。

(b) 计算变量。利用TRD_Mnth和FS_Combas，生成数据BM：计算每个公司在每年的12月31日的权益账面价值与市场价值的比值（b/m）；生成数据SIZE：保留每个公司在每年4月30日的市值cap（来自TRD_Mnth）。

(c) 合并操作。对每个股票，合并本年cap和上一年B/M形成新数据FACTOR。

(d) 分组和编码变量。对FACTOR数据按照年分组，每一年先按照公司cap大小平均分成2组small和big，再分别对这2组根据B/M从小到达分成3组，即前30%为value，中间40%为neutral，后30%为growth。这样共有6组。

(e) 条件匹配操作。匹配TRD_Mnth和FACTOR，生成新数据FF。对TRD_Mnth每一行数据，如果交易日期在当年的1月1日--4月31日，匹配前年底的FACTOR数据；如果交易日期在当年的5月1日--12月31日，匹配去年底的FACTOR数据。

(f) 描述统计。对数据FF按照月-组合号分组，计算平均回报率。再按照月分组，利用计算出来的平均回报率计算变量smb和hml：

_smb = 1/3 (Small Value + Small Neutral + Small Growth)-1/3 (Big Value + Big Neutral + Big Growth)_

_hml = 1/2 (Small Value + Big Value)-1/2 (Small Growth + Big Growth)_

(g) 画图。在一幅图上画出smb和hml的时间序列图。


代码如下：

```sas
libname csmar ("F:\CSMAR\TR" "F:\CSMAR\FS");   /* my data dir */

/* 需要用到的数据集来自CSMAR：
	Trd_mnth：月个股交易数据
	Fs_combas：资产负债表数据
	Trd_nrrate：无风险利率数据
*/
data msf (label="月个股交易数据" drop=Markettype);
	set csmar.trd_mnth (where=(Markettype in (1,4,16))
						keep=stkcd Trdmnt Markettype Mretwd Msmvosd);
	Msmvosd=Msmvosd*1000;            /* CSMAR数据中市值和交易量的单位是千元 */
	if nmiss(stkcd,Trdmnt,Mretwd,Msmvosd)=0;
	label stkcd="证券代码"
	      Trdmnt="交易月份，用每月最后一个日历日期表示"
		  Mretwd="收益率"
		  Msmvosd="月末市值"
		  ;
run;


data bs (label="资产负债表数据");
	set csmar.Fs_combas (keep=stkcd Accper A003000000 Typrep); 
	where month(Accper)=12 and Typrep='A' and 
			nmiss(stkcd,Accper,A003000000)=0; 
	rename A003000000=ceq;
	label stkcd="证券代码"
	      Accper="会计期间"
		  ceq="所有者权益"
		  ;
	run;

data rf (label="月度无风险收益" drop=Nrr1);
	set csmar.TRD_Nrrate (where=(Nrr1='NRI01') keep=Clsdt Nrrmtdt Nrr1);
	yr=year(Clsdt);
	mon=month(Clsdt);
	Nrrmtdt=Nrrmtdt/100;       /* 单位转化，和后面的股票收益单位一致 */
	run;

proc sort data=rf; by Clsdt; run;

data rf;
	set rf;
	by yr mon;
	if last.mon;
	run;

data rf (keep=Clsdt Nrrmtdt);
	set rf;
	Clsdt=intnx('month',Clsdt,0,'E');    /* 为防止日期数据没有对齐到月末 */
	run;


* 计算size和BM因子，形成公司-年数据;
proc sql;
	create table bm as
	select a.stkcd, a.trdmnt, b.ceq/a.Msmvosd as bm "b/m ratio" from msf a
		inner join bs b
	on a.stkcd=b.stkcd and a.trdmnt=b.accper;
	quit;

data size;
	set msf;
	where month(trdmnt)=4;
	run;

proc sql;
	create table factor as
	select a.*, b.Msmvosd as cap "market cap." from bm a
		inner join size b
	on a.stkcd=b.stkcd and a.Trdmnt=intnx('month', b.Trdmnt, -4, 'E');
	quit;


/* 将月度股票交易数据和因子合并起来:
		如果交易日期在当年的1月1日--4月31日，匹配前年底的FACTOR数据；
		如果交易日期在当年的5月1日--12月31日，匹配去年底的FACTOR数据
*/

proc sql;
	create table ff3f_1 as 
	select a.*, b.cap, b.bm from msf (where=(month(Trdmnt)<=4)) a 
		inner join factor b
	on a.stkcd=b.stkcd and intnx('year', a.trdmnt, -2,'E')=b.Trdmnt;
quit;

proc sql;
	create table ff3f_2 as 
	select a.*, b.cap, b.bm from msf (where=(month(Trdmnt)>4)) a 
		inner join factor b
	on a.stkcd=b.stkcd and intnx('year', a.trdmnt, -1,'E')=b.Trdmnt;
quit;

data ff3f;
	set ff3f_1 ff3f_2;
	run;

/*  依据FF （1992）和FF（1993）的方法：
		每月，根据size分成2个组合，在每个size组合内，再根据bm分成3个组合；
		所以分组是有顺序的
*/
proc sort data=ff3f; by Trdmnt cap bm; run;

proc rank data=ff3f out=ff_port groups=2;
	by Trdmnt;
	ranks size_port;
	var cap;
	run;

proc sort data=ff_port; by Trdmnt size_port; run;

proc rank data=ff_port out=ff_port groups=10;
	by Trdmnt size_port;
	ranks bm_port;
	var bm;
	run;

data ff_port;     /* 将size组合标号为（1,2），将bm组合编号为（1,2,3） */
	set ff_port;
	size_port=size_port+1;
	bm_port=(bm_port<=2)+2*(3<=bm_port<=6)+3*(7<=bm_port<=9);
	run;


/*  计算因子组合的收益率，从而建立了月度的FF因子收益率。
		这部分考虑了等权重的因子收益和按照市值加权的因子收益
*/


* 市场因子;
proc sql;
	create table mkt_ret as
	select  trdmnt, 
			mean(Mretwd) as mkt_ewret "等权重的市场收益",
			sum(Mretwd*Msmvosd)/sum(Msmvosd) as mkt_vwret "按照市值加权的市场收益" 
			from ff_port
	group by trdmnt;
	quit; 

proc sql;
	create table mkt_exret as
	select a.trdmnt, a.mkt_ewret-b.Nrrmtdt as mkt_ew, a.mkt_vwret-b.Nrrmtdt as mkt_vw 
		from mkt_ret a 
	inner join rf b
	on a.trdmnt=b.Clsdt;
	quit; 

* size和bm因子收益;

proc sql;
	create table ff_port_ret as
	select  Trdmnt, size_port, bm_port, mean(Mretwd) as ewret,
			sum(Mretwd*Msmvosd)/sum(Msmvosd) as vwret from ff_port
	group by Trdmnt, size_port, bm_port;
quit;

proc transpose data=ff_port_ret out=ff_port_ret1;
	by Trdmnt;
	id size_port bm_port;
	var ewret vwret;
run;

data ff_port_ret2;
	set ff_port_ret1;
	smb=(_11+_12+_13)/3-(_21+_22+_23)/3;
	hml=(_11+_21)/2-(_13+_23)/2;
	keep Trdmnt _name_ smb hml;
	run;

data ff_port_ret3 (drop=_name_);
	merge ff_port_ret2 (where=(_name_='ewret')  
						rename=(smb=smb_ew hml=hml_ew)) 
		ff_port_ret2 (where=(_name_='vwret') 
						rename=(smb=smb_vw hml=hml_vw));
	run;


* 3因子收益，将时间区间限制在[2000/1/1-2016/12/31];

data ff_ret;
	merge ff_port_ret3 mkt_exret;
	where '01jan2000'd <=Trdmnt<='31dec2016'd;
	run;



/*  画图：因子组合的收益率
	描述统计:
*/
proc sgplot data=ff_ret;
	series x=Trdmnt y=mkt_ew;
	series x=Trdmnt y=smb_ew;
	series x=Trdmnt y=hml_ew;
run;

proc sgplot data=ff_ret;
	series x=Trdmnt y=mkt_ew;
	series x=Trdmnt y=mkt_vw;
run;

proc sgplot data=ff_ret;
	series x=Trdmnt y=smb_ew;
	series x=Trdmnt y=smb_vw;
run;

proc sgplot data=ff_ret;
	series x=Trdmnt y=hml_ew;
	series x=Trdmnt y=hml_vw;
run;

/* 除了等权重的bm因子收益外，其余因子在时间序列上的平均收益均为正，符合预期。
	简单(naive) t-test显示，中国市场bm因子并不显著，市场因子和规模因子显著。
*/

proc means data=ff_ret mean std median p25 p75 t probt;
	var smb_ew smb_vw hml_ew hml_vw mkt_ew mkt_vw;
	run;

```



## 2.	事件研究数据处理

这里的事件研究分析是从Dellavigna and Pollet (2009)中抽取的主要结果，检验不同类型的盈余公告（是否在星期五发布盈余公告）是否具有不同的短期和中长期效应(PEAD)。

(a) 利用Trd_Index建立交易日所对应的日历日期，生成交易日历数据CALENDAR。对数据CALENDAR按照日期从小到大排序，生成一列整数index对应日期所在的行号。这样做是为了标记交易日的顺序。

(b) 对IAR_Rept保留公司和公告日2个变量。将IAR_Rept和CALENDAR合并，如果公告日在非交易日，选择之后最近的交易日匹配。

(c) 对每个事件，选择估计窗口[-280, -34]，用Fama- French三因子模型估计系数。

(d) IAR_Rept中每个盈余公告事件，从TRD_Dalyr选择事件[0,75]窗口的股票交易数据，计算每个事件的BHAR[0, i](其中i=1-75)，选用市场模型作为基准收益

(e) 利用AF_Forecast，计算盈余公告日前1年内分析师对EPS预测的平均值，计算earnings surprise (SUE) 

(f) 每一年，按照盈余公告类型分为2类：周末／工作日公告盈余，在每种盈余公告下再根据SUE从低到高分成10个投资组合。计算每个组合的BHAR[0,1]和BHAR[2,75]

(g) 在事件序列上对每个组合的BHAR[0,1]和BHAR[2,75]平均，计算BHAAR[0,1]和BHAAR[2,75]

(h)画图展示BHAAR[0,1]和BHAAR[2,75]在不同组合上的差异。中国数据是否能够产生Dellavigna and Pollet (2009)的结果。



### 事件研究的背景资料（非技术）

事件研究是非常重要的方法论（methodology），为什么要做事件研究，请见 [The basic idea and purpose of event studies](http://eventstudymetrics.com/index.php/the-basic-idea-and-purpose-of-event-studies/)

在写代码前要先明白事件研究的基本概念，请见[Event Study Methodology](http://eventstudymetrics.com/index.php/event-study-methodology/)。

1. Timeline
    - estimation window
    - event window

2. Benchmark Model
    - matched firms model (DGTW 1997)
    - market return model
    - constant mean return model
    - Fama French 3 factor model
    
3. Estimator
    - AR (AAR)
    - CAR (CAAR)
    - BHAR (BHAAR)
    
4. Significance Tests 
    - cross-sectional t-test
    - standardized residual test (Patell, 1976)
    - standardized cross-sectional test statistic (Boehmer, Musumeci and Poulsen, 1991)
    - non-parametric rank test by Corrado (1989)
    - generalized sign test
    - skewness-adjusted t-test 

事件研究可根据事件窗口的长短分为短窗口的事件研究和长窗口的事件研究，短窗口一般在一月以内，长窗口一般在半年至5年。如果价格能够迅速反应一般用短窗口，而评价公司行为（如IPO, SEO, M&A）的长期绩效往往用长窗口，请见[Nine steps to follow when performing a short-term event study](http://eventstudymetrics.com/index.php/9-steps-to-follow-when-performing-a-short-time-study/)。短窗口的事件研究在统计推断方面问题不大，但是长窗口的事件研究比较麻烦。对长窗口在构建资产组合方面有2种方法buy-and-hold returns和calendar-time portfolios，请见[Long-term event studies and the calender-time portfolio approach](http://eventstudymetrics.com/index.php/long-term-event-studies-and-the-calender-time-portfolio-approach/)。

#### 事件研究资料（个别资料有需要对计量有一定理解）

综述事件研究比较经典的资料是MacKinlay (1997)，Kothari and Warner (2008)，Seppo Pynnönen的[notes](http://lipas.uwasa.fi/~sjp/articles/sp_acta_wasaensia_143_327-354.pdf)。对长窗口的事件研究的经典论文Barber and Lyon (1997), Kothari and Warner (1997), Lyon, Barber and Tsai (1999)。基准的资产定价模型可参考Fama and French (1993)和DGTW (1997)。


代码如下：

```sas
/* 用中国A股数据复制 Dellavigna, S., & Pollet, J. M. (2009). 
Investor inattention and friday earnings announcements. 
The Journal of Finance, 64(2), 709–749. */


/*用到的数据集，来自CSMAR：
	IAR_Rept：盈余公告日期 
	Trd_Index：指数交易数据 
	dsf：个股交易数据 
	af_forecast：分析师预测数据 
	af_actual：实际盈余 
	Stk_mkt_thrfacday：日度三因子数据 */

* 输入数据和输出数据;  
libname csmar ( "F:\CSMAR\AF"  "F:\CSMAR\FAnn"  "F:\CSMAR\TR" );  /* my data dir */
libname temp "C:\Users\wang123\Desktop\temp";


* 盈余公告日期，2007-2015年年报 ;
data FAnn (drop=Reptyp label='盈余公告日期（只包括2006-2015年年报）');
	set csmar.IAR_Rept (keep=stkcd accper Annodt Reptyp
		where=(Reptyp=4 and '01Jan2007'd<=accper<='31Dec2015'd));
	run;

* 建立交易日期的索引，‘2006-01-04’标记为1，下一个交易日标记为2，以此类推 ;
data tr_calendar (label='构建交易日历：将交易日和日历日建立1-1对应' drop=indexcd);
 	set csmar.Trd_Index (where=(indexcd=1 and '01Jan2005'd<=Trddt<='31Dec2016'd)
						 keep=Trddt indexcd);
run;

proc sort data=tr_calendar; by Trddt; run;

data tr_calendar;
	set tr_calendar;
	tr_index+1;
	run;

* 确定事件日;
* 如果盈余公告日是交易日日，以盈余公告日作为事件日；
  如果盈余公告日在非交易日期，则取下一个交易日期为事件日，如果下一个交易日发生距公告日不超过5天 ;
proc sql;  /* cost 53 secs */
	create table evt_date as 
	select a.stkcd, a.accper, a.annodt, b.trddt, b.tr_index from fann a
		left join tr_calendar b
	on a.annodt<=b.trddt<=a.annodt+5
	group by a.stkcd, a.annodt
	having b.trddt-a.annodt=min(b.trddt-a.annodt);
	quit;

data evt_date (label='事件时期数据');
	set evt_date;
	rename Trddt=evtdate;
	label  
		Stkcd='股票代码'
		Trddt='事件日期'
		Accper='会计截止日期'
		tr_index='自2005年以来第i个交易日'
		Annodt='报告公布日期';
run;

* 日度股票交易数据，剔除B股数据 ;
data dsf (drop=Markettype label='日各股交易数据');
	set csmar.dsf (keep=stkcd Trddt Clsprc Dretwd Markettype
		where=('01Jan2005'd<=Trddt<='31Dec2016'd and Markettype in (1,4,16) and 
				nmiss(stkcd, Trddt, Clsprc, Dretwd, Markettype)=0 and 
				dretwd <0.11 and dretwd >-0.11));
	label
		stkcd="证券代码"
		Trddt="交易日期"
		Clsprc="日收盘价"
		Dretwd="考虑现金红利再投资的日个股回报率";
	run;


* 分析师预测数据过滤，形成公司-年-分析师数据：
	（1）只保留盈余公告日最近12个月的盈利预测，并且
	（2）如果一个分析师有多项预测，则仅保留最近的一次预测 ;
proc sql;
	create table af_forecast as
	select a.stkcd "股票代码", a.Fenddt "会计截止日期", a.Feps "EPS预测"
		from csmar.af_forecast(where=('01Jan2006'd<=Fenddt<='31Dec2015'd)) a 
	inner join evt_date b
	on a.stkcd=b.stkcd and a.fenddt=b.accper and 
		intnx('month', b.Annodt, -12, 'B')<a.Rptdt<b.Annodt
	where nmiss(a.stkcd, a.Fenddt, a.Feps)=0
	group by a.stkcd, a.Fenddt, a.AnanmID
		having a.Rptdt=max(a.Rptdt);
quit;


* EPS一致预测（EPS consensus forecast），公司-年数据;
proc sql;
	create table eps_cons as  
	select stkcd, Fenddt, median(Feps) as eps_cons from af_forecast
		group by stkcd, Fenddt;
	quit;


data af_actual (label='实际公告的EPS');
	set csmar.af_actual (keep= stkcd Ddate Meps
		where=('01Jan2006'd<=ddate<='31Dec2015'd));
	run;

* 盈余公告日前5个交易日收盘价，作为SUE的分母;
proc sql;
	create table prc as
	select a.stkcd, a.accper, b.Clsprc as prc '基准收盘价'
	from evt_date a 
		inner join Tr_calendar c
	on a.tr_index-5=c.tr_index   
		inner join dsf b
	on a.stkcd=b.stkcd and b.Trddt=c.Trddt;
	quit;

* 盈余惊喜（earnings surprise）;
proc sql;
	create table sue as
	select a.stkcd, a.fenddt, (a.eps_cons-b.Meps)/c.prc as sue "盈余惊喜" from eps_cons a
		inner join af_actual b
	on a.stkcd=b.stkcd and a.fenddt=b.ddate
		inner join prc c
	on a.stkcd=c.stkcd and a.fenddt=c.Accper
	where not missing(calculated sue);
	quit;


* 事件研究估计窗口选择，取[-280, -34];
proc sql;
	create table pre_event as 
	select a.stkcd, a.accper, b.Dretwd,
		   d.RiskPremium1 as rm, d.SMB1 as smb, d.HML1 as hml from evt_date a
	left join tr_calendar c
		on a.tr_index-280 <=c.tr_index<=a.tr_index -34
	left join dsf b
		on b.Trddt=c.Trddt and b.stkcd=a.stkcd
	left join csmar.Stk_mkt_thrfacday (keep=TradingDate RiskPremium1 SMB1 HML1 MarkettypeID   
									   where=(MarkettypeID='P9709')) d
		on b.Trddt=d.TradingDate
	where nmiss(Dretwd, rm, smb, hml)=0
	order by stkcd, accper;
quit; 


/* 对每个事件，计算三因子模型的因子载荷 */
proc printto log=junk; run;
proc reg data=pre_event noprint outest=ff_est edf;
	by stkcd accper;
	model Dretwd = rm smb hml;
	quit;
proc printto; run;

data ff_est;
	set ff_est;
	obs=_EDF_+_P_;
	keep stkcd accper Intercept rm smb hml obs _RMSE_; 
	if obs>120;			/* 要求估计窗口的交易日期至少达到120天 */
	label Intercept="截距项估计值"
		  rm="市场因子系数"
		  smb="市值因子系数"
		  hml="价值因子系数"
		  obs="估计所用样本数目"
		  _RMSE_="标准误（se）";
	run;


* 利用事件前窗口估计所得系数，建立基准资产组合的收益率
	估计[0,75]的AR, CAR, BHAR;
proc sql;
create table post_event as
    select a.stkcd, a.accper, a.annodt, a.evtdate,
		   c.tr_index-a.tr_index as re_date "相对于事件日的相对交易日", b.Dretwd, 
		   e.Intercept+e.rm*d.RiskPremium1+e.smb*d.SMB1+e.hml*d.HML1 
		   as ret_bench "由三因子模型确定的基准收益率" from evt_date a
	left join tr_calendar c
		on a.tr_index <=c.tr_index<=a.tr_index +75
	left join dsf b
		on b.Trddt=c.Trddt and b.stkcd=a.stkcd
	left join csmar.Stk_mkt_thrfacday (keep=TradingDate RiskPremium1 SMB1 HML1 MarkettypeID   
									   where=(MarkettypeID='P9709')) d
		on b.Trddt=d.TradingDate
	left join ff_est e
		on a.stkcd=e.stkcd and a.accper=e.accper
	where nmiss(Dretwd, calculated ret_bench)=0
	order by stkcd, accper, re_date;
quit; 

data evt_stat (keep=stkcd accper re_date bhar label='时间研究所得的异常收益，个股层面');
	set post_event;
	by stkcd accper;
	ar=Dretwd-ret_bench;
	retain cumret cumbenchret car;
	cumret=cumret*(1+Dretwd);
	cumbenchret=cumbenchret*(1+ret_bench);
	car=car+ar;
	if first.accper then do;
		car=ar;
		cumret=1+Dretwd;
		cumbenchret=1+ret_bench;
	end;
	bhar=cumret-cumbenchret;
run;


data evt_car (label="股价的短期和长期反应"
			   where=(nmiss(bhar_s, bhar_all)=0));
	merge evt_stat (rename=(bhar=bhar_s) where=(re_date=1))
		  evt_stat (rename=(bhar=bhar_all) where=(re_date=75));
	by stkcd accper;
	run;

data evt_car;
	set evt_car;
	bhar_l=(bhar_all+1)/(bhar_s+1)-1;
	label bhar_s="股价的短期反应BHAR[0, 1]" 
          bhar_all="股价的全部反应[0,75]"
		  bhar_l="股价的短期反应BHAR[2, 75]";
	run;

* 和原文不同，我们对比在周五、周六(我们样本中周日没有公告)相对于周内公告的情况
	因为美国在周末几乎没有公告，国内周六公告仍然较多;
proc sql;
	create table evt_car as
	select a.*, weekday(b.Annodt)-1 as dayinweek "用1-7代指周一至周日", 
		weekday(b.Annodt) in (1, 6, 7) as weekend "周末公告的虚拟变量",
		c.sue from evt_car a 
	inner join fann b on a.stkcd=b.stkcd and a.accper=b.accper
	inner join sue c on a.stkcd=c.stkcd and a.accper=c.fenddt;
quit; 

proc freq data=evt_car;
	tables dayinweek  accper*weekend / nopercent nocol norow nocum;
run; 


* 在每年，分别对“周末公告组”和“周内公告组”按照盈余惊喜从小到大分成10组;
proc sort data=evt_car; by accper weekend; run;

proc rank data=evt_car groups=10 out=car_by_sue;
	by accper weekend;
	var sue;
	ranks sue_rank;
	run;

data car_by_sue (label='按照SUE分组的盈余公告异常收益');
	set car_by_sue;
	sue_rank+1;
	run;

proc sql;
	create table caar as 
	select weekend, sue_rank "SUE组合，10为盈余最高",
		mean(bhar_s) as bhaar_s "每组平均BHAR[0,1]", 
		mean(bhar_l) as bhaar_l "每组平均BHAR[2,75]",
		mean(bhar_all) as bhaar_all "每组平均BHAR[0,75]",
		mean(sue) as sue_avg "每组平均SUE"
	from car_by_sue
	group by weekend, sue_rank;
	quit; 


* 画图pp721-722;

proc sgplot data=caar;
	series x=sue_rank y=bhaar_s / group=weekend;
	refline 0 / axis=x;
run;

proc sgplot data=caar;
	series x=sue_rank y=bhaar_l / group=weekend;
run;

proc sgplot data=caar;
	series x=sue_avg y=bhaar_l / group=weekend;
run;
```




## REFERENCE



Barber, B.M. and Lyon, J.D. (1997): Detecting Long-Run Abnormal Stock Returns: Empirical Power and Specification of Test-Statistics, _Journal of Financial Economics_ 43, 341-372.

Daniel, K., Grinblatt, M., Titman, S., and Wermers, R. (1997). Measuring Mutual Fund Performance with Characteristic-Based Benchmarks. _Journal of Finance_, 52(3), 1035-1058.

Dellavigna, S. and Pollet, J. M. (2009). 
Investor inattention and friday earnings announcements.  
_The Journal of Finance_, 64(2), 709–749.

Fama, E.F. and French, K.R. (1993): Common Risk Factors in the Returns on Stocks and Bonds, _Journal of Financial Economics_ 33, 3-56.

Kothari, S.P. and Warner, J.B. (1997): Measuring Long-Horizon Security Price Performance, _Journal of Financial Economics_ 43, 301-340.

Kothari, S.P. and Warner, J.B. (2008): Econometrics of Event Studies, in: Eckbo, B.E. (ed.), _Handbook of Corporate Finance: Empirical Corporate Finance_, Vol. 1, Elsevier/North-Holland, 3-36.

Lyon, J. D., Barber, B.D. and Tsai, C. (1999): Improved Methods for Tests of Long-Run Abnormal Stock Returns, _Journal of Finance_ 54, 165-201.

MacKinlay, C.A. (1997): Event Studies in Economics and Finance, _Journal of Economic Literature_ 35, 13-39.

