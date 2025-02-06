# Stata-
储存本科毕业论文“青年生育意愿”Stata代码
//处理个人数据
use "/Users/hemingjing/Desktop/CFPS/cfps2018/cfps2018person_202012.dta",clear

*计算受访者参与养老保险项目的个数
forval i=1/7{
recode qi301_a_`i' (1=1)(0=0)(else=.),gen(qi301_`i')
label var qi301_`i' "trial"
}

egen 参与养老保险个数=rowtotal(qi301_1-qi301_7)
label var 参与养老保险个数 "参与养老保险个数"

keep pid fid18 qka202 qf703_a_1 qf703_a_2 qf603_a_1 qf603_a_2 gender urban18 age w01 参与养老保险个数
rename qka202 b_will
rename w01 edu
rename urban18 urban

*研究对象为年龄在18~35岁之间的青年
keep if age>=18 & age<=35 //drop后还有10136个样本

*生成照料支持变量
replace qf703_a_1=0 if qf703_a_1==-8
replace qf703_a_2=0 if qf703_a_2==-8
gen 照料支持=qf703_a_1+qf703_a_2
label var 照料支持 "照料支持程度"

*生成赡养负担变量
replace qf603_a_1=0 if qf603_a_1==-8
replace qf603_a_2=0 if qf603_a_2==-8
gen 赡养负担=qf603_a_1+qf603_a_2
label var 赡养负担 "赡养负担程度"

*处理缺失值
drop if b_will<0
drop if urban==-9
drop if edu<0 //drop后还有8672个样本

egen mis = rmiss(_all)
drop if mis>0
drop mis //drop后还有8331个样本

*清洗“最高学历”数据
gen 受教育程度=0 if edu==10 //没上过学
replace 受教育程度=1 if edu==0 //文盲或半文盲
replace 受教育程度=2 if edu==3 //小学
replace 受教育程度=3 if edu==4 //初中
replace 受教育程度=4 if edu==5 //高中、中专、技校、职高
replace 受教育程度=5 if edu==6 //大专
replace 受教育程度=6 if edu==7 //大学本科
replace 受教育程度=7 if edu==8 //硕士
replace 受教育程度=8 if edu==9 //博士
drop edu
label var 受教育程度 "最高学历"

*保存个人信息数据
save "/Users/hemingjing/Desktop/论文_1.dta",replace


//处理个人-家庭数据
use "/Users/hemingjing/Desktop/CFPS/cfps2018/cfps2018famconf_202008.dta"

keep pid fid18 fid_provcd18 tb3_a18_p pid_a_f pid_a_m

*清洗省份数据，得到“地区”变量
gen region_west=1 if inlist(fid_provcd18,50,51,52,53,54,61,62,63,64,65)
replace region_west =0 if region_west==.
label var region_west "西部地区"

gen region_east=1 if inlist(fid_provcd18,11,12,13,21,31,32,33,35,37,44,46)
replace region_east =0 if region_east==.
label var region_east "东部地区"

gen region_mid=1 if inlist(fid_provcd18,14,15,22,23,34,36,41,42,43,45)
replace region_mid =0 if region_mid==.
label var region_mid "中部地区"

gen region=1 if region_west==1
replace region=2 if region_mid==1
replace region=3 if region_east==1
rename region 地区
label var 地区 "所属地区"

*处理个人婚姻状态变量
drop if tb3_a18_p==-8
gen 婚姻状态=1 if inlist(tb3_a18_p,2,3) //"在婚"或“同居”算作已婚
replace 婚姻状态=0 if 婚姻状态==. //“未婚”、“离婚”或“丧偶”算作未婚
label var 婚姻状态 "婚姻状态"

*保存个人-家庭数据
save "/Users/hemingjing/Desktop/论文_2.dta",replace 

//再次使用个人-家庭数据库，利用母亲的pid进行merge，得到兄弟姐妹数量的变量
use "/Users/hemingjing/Desktop/CFPS/cfps2018/cfps2018famconf_202008.dta"

forval i=1/10{
recode alive_a18_c`i' (1=1)(0=0)(else=.),gen(alive_c`i')
label var alive_c`i' "trial"
}

egen 母亲子女数=rowtotal(alive_c1-alive_c10)
label var 母亲子女数 "母亲的子女个数"
rename pid pid_a_m
label var pid_a_m "母亲样本编码"

keep pid fid18 母亲子女数
save "/Users/hemingjing/Desktop/母亲的子女个数.dta",replace

*利用pid_a_m和fid18匹配上面的两个数据库
use "/Users/hemingjing/Desktop/论文_2.dta"
merge m:1 pid_a_m fid18 using "/Users/hemingjing/Desktop/母亲的子女个数.dta"
keep if _merge==3
drop _merge

*按照逻辑，母亲应该至少有编码为pid的个体这一个孩子
drop if 母亲子女数<1 //drop后有15338个数据

gen 兄弟姐妹数量= 母亲子女数-1
label var 兄弟姐妹数量 "个人的兄弟姐妹数量"

*保存个人-家庭数据
save "/Users/hemingjing/Desktop/论文_2.dta",replace

//得到家庭经济情况数据
use "/Users/hemingjing/Desktop/CFPS/cfps2018/cfps2018famecon_202101.dta"

keep fid18 finc fincome1_per fincome1_per_p mortage

drop if finc<0
gen family_income= finc/12
label var family_income "家庭月均收入"
gen 家庭收入=ln(1+family_income)
label var 家庭收入 "家庭收入变量"

*保存家庭经济数据库
save "/Users/hemingjing/Desktop/论文_3.dta",replace


//匹配各数据库
use "/Users/hemingjing/Desktop/论文_1.dta"

*匹配个人库和家庭经济信息库
merge m:1 fid18 using "/Users/hemingjing/Desktop/论文_3.dta"
keep if _merge==3
drop _merge //匹配后还有8050个样本

save "/Users/hemingjing/Desktop/论文_1.dta",replace 

*匹配个人库和个人-家庭库
merge 1:1 pid fid18 using "/Users/hemingjing/Desktop/论文_2.dta"
keep if _merge==3
drop _merge //匹配后还有3899个样本
 

*调整变量顺序
order pid fid18 b_will 照料支持 赡养负担 受教育程度 age 婚姻状态 兄弟姐妹数量 参与养老保险个数 家庭收入 urban 地区, first

*对变量重新命名
rename b_will 生育意愿
rename age 年龄
rename urban 城乡户口

save "/Users/hemingjing/Desktop/论文_1.dta",replace

//描述性统计及回归结果
*描述性统计
local varlist "生育意愿 赡养负担 照料支持 受教育程度 年龄 婚姻状态 兄弟姐妹数量 参与养老保险个数 家庭收入 城乡户口 地区"
estpost summarize `varlist', detail
esttab using Myfile.rtf, cells("count mean(fmt(2)) sd(fmt(2)) min(fmt(2)) p50(fmt(2)) max(fmt(2))") noobs compress replace title(esttab_Table: Descriptive statistics)

*ologit回归
ologit 生育意愿 赡养负担
eststo column1
ologit 生育意愿 照料支持
eststo column2
ologit 生育意愿 赡养负担 照料支持
eststo column3
ologit 生育意愿 赡养负担 照料支持 受教育程度 年龄 兄弟姐妹数量 参与养老保险个数 家庭收入 城乡户口 地区
eststo column4
outreg2 [column1 column2 column3 column4] using "/Users/hemingjing/Desktop/essay.xls", ctitle(b_will) stats(coef,se) bdec(3) sdec(3) excel replace

****决定不放婚姻状态变量，因为结果很糟糕


*OLS回归
reg 生育意愿 赡养负担
eststo column1
reg 生育意愿 照料支持
eststo column2
reg 生育意愿 赡养负担 照料支持
eststo column3
reg 生育意愿 赡养负担 照料支持 受教育程度 年龄 兄弟姐妹数量 参与养老保险个数 家庭收入 城乡户口 地区
eststo column4

*替换被解释变量为二孩生育意愿
logit 二孩生育意愿 赡养负担程度 照料支持程度 受教育程度 年龄 兄弟姐妹数量 参与养老保险个数 家庭收入 城乡户口 地区 
eststo column5

*输出结果
outreg2 [column1 column2 column3 column4] using "/Users/hemingjing/Desktop/essay2.xls", ctitle(稳健性检验) stats(coef,se) bdec(3) sdec(3) excel replace


**异质性分析
*是否接受过高等教育
ologit 生育意愿 赡养负担 照料支持 受教育程度 年龄 兄弟姐妹数量 参与养老保险个数 家庭收入 城乡户口 地区 if old_var==0
eststo column1 //未接受过高等教育

ologit 生育意愿 赡养负担 照料支持 受教育程度 年龄 兄弟姐妹数量 参与养老保险个数 家庭收入 城乡户口 地区 if old_var==1
eststo column2 //接受过高等教育

*年龄是否在30~35岁之间
ologit 生育意愿 赡养负担 照料支持 受教育程度 年龄 兄弟姐妹数量 参与养老保险个数 家庭收入 城乡户口 地区 if new_var==0
eststo column3 //年龄在18~29岁

ologit 生育意愿 赡养负担 照料支持 受教育程度 年龄 兄弟姐妹数量 参与养老保险个数 家庭收入 城乡户口 地区 if new_var==1
eststo column4 //年龄在30~35岁

outreg2 [column1 column2 column3 column4] using "/Users/hemingjing/Desktop/essay3.xls", ctitle(b_will) stats(coef,se) bdec(3) sdec(3) excel replace

//机制检验
use "/Users/hemingjing/Desktop/备胎数据_2.dta"

ologit qm509 赡养负担程度 受教育程度 年龄 sib_num 参与养老保险个数 家庭收入 城乡户口 地区
eststo column1 //传宗接代观念

logit qm6015 赡养负担程度 受教育程度 年龄 sib_num 参与养老保险个数 家庭收入 城乡户口 地区
eststo column2 //是否相信祖先

ologit qm510 赡养负担程度 受教育程度 年龄 sib_num 参与养老保险个数 家庭收入 城乡户口 地区
eststo column3 //子女有出息

ologit qm507 赡养负担程度 受教育程度 年龄 sib_num 参与养老保险个数 家庭收入 城乡户口 地区
eststo column4 //死后有人念想

outreg2 [column1 column2 column3 column4] using "/Users/hemingjing/Desktop/essay4.xls", ctitle(b_will) stats(coef,se) bdec(3) sdec(3) excel replace
