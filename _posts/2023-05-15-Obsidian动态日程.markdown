---
tag: 分享
create: 2023-05-14 15:36
---
![2 拷贝.png|1400](https://cwl-img.oss-cn-beijing.aliyuncs.com/202305141811205.png)
## 可以做到什么？
1. 创建每天精确时间的固定日程任务
2. 自动在配置的每周几时创建日程任务
3. 自动在配置的每月几日创建日程任务
4. 匹配排期任务，指定日期在未来自动创建对应日程任务
5. 通过逻辑判断创建任务
6. 生成的周记动态统计本周每天的日程内容

## 需要插件
- **Templater** - 提供脚本模板以及脚本函数支持
- **Dataview** - 提供脚本函数支持
- *Calendar* - 右侧日历
- *Day Planner* - 右侧日程表
- *Checklist* - 右侧任务清单
- ToggleList - 快捷切换任务状态
- Style Setting - 样式美化


## 如何使用
### 存放模板
在你的笔记中创建一个文件夹专门用于存储***模板***，并且创建一个***日记***文件夹和一个***周记***文件夹，位置不限。
在*模板文件夹*中新建两个文件，放入下面分享的**日记模板**和**周记模板**。

````MARKDOWN TI:"日记模板"
<%*
	//今天日期
	let today = new Date();
	//预输出变量
	let taskListStr = "";
	
	//时间段
	const timeRegionArray = [
		"00:00|04:59|凌晨",
		"05:00|08:59|早晨",
		"09:00|11:29|上午",
		"11:30|13:29|中午",
		"13:30|17:59|下午",
		"18:00|23:59|晚上",
	];
	//每日任务清单，预输入固定的每日任务
	let taskArray = [
		"07:20|🌞起床",
		"07:25|🪥喝一杯水，洗漱",
		"07:30|🍵**泡茶**",
		"12:25|👀眼保健操",
		"12:30|💤午睡",
		"21:30|📚自考学习",
		"23:15|🥛热牛奶",
		"23:50|🌙洗漱，早睡",
	];
	//周常任务清单
	let weekTaskArray = [{},[],[],[],[],[],[],[]];
	//想把任务加在周几，数字就填几
	weekTaskArray[5].push("17:00|📰**周报**");
	weekTaskArray[6].push("16:00|📞**给妈妈打电话**");
	weekTaskArray[7].push("17:00|🧹*整理环境*");
	weekTaskArray[7].push("22:00|🗒️**周记**");
	//周常任务添加
	taskArray.push.apply(taskArray,weekTaskArray[(today.getDay() == 0 ? 7 : today.getDay())]);

	//月任务清单
	let monthTaskArray = [{},[],[],[],[],[],[],[],[],[],[],[],[],[],[],[],[],[],[],[],[],[],[],[],[],[],[],[],[],[],[],[]];
	//想把任务加在每月几号，数字就填几
	monthTaskArray[10].push("17:00|🗒💴发工资咯~");
	//月任务添加
	taskArray.push.apply(taskArray,monthTaskArray[today.getDate()]);

	//排期计划文件
	const planFile = tp.file.find_tfile("排期计划");
	//生成日程后排期计划内当天的任务是否删除(true=删除，false=不删除)
	const planFileRemoveFlg = true;
	//排期计划中未指定精确时间的任务应该在哪个时间点
	const planDefaultTaskTime = "09:00";

	//动态任务，需要独立判断
	//昨天日期
	let yesterdayDate = new Date(today);
	yesterdayDate.setDate(today.getDate() - 1);
	// 昨天日记文件名
	let fullFileName = "日记："+yesterdayDate.getFullYear()+"年"+(yesterdayDate.getMonth()+1)+"月"+yesterdayDate.getDate()+"日";
	const file = tp.file.find_tfile(fullFileName);
	if(file != null){
		const content = await app.vault.cachedRead(file);
		const contentStr = content.toString();
		// 隔一天跑步散步
		if(contentStr.contains("- [x] 20:30 👟跑步")){
			taskArray.push("20:30|👞散步");
		}else{
			taskArray.push("20:30|👟跑步");
		}
	}

	//排期计划检索
	let planTask = "";
	if(planFile != null){
		const content = await app.vault.cachedRead(planFile);
		//分行
		const lineArray = content.split("\n");
		const dateTitle = today.getFullYear()+"年"+(today.getMonth()+1)+"月"+today.getDate()+"日";
		let modifyLabel = [];
		for(var i = 0;i < lineArray.length;i++){
			const nowLine = lineArray[i];
			if(nowLine.startsWith("## " + dateTitle)){
				modifyLabel.push(nowLine);
				//循环这天标签下的任务
				for(var j = 1;j < lineArray.length - i;j++){
					let lineTask = lineArray[i+j];
					//碰到空白行和#开头则退出循环
					if(lineTask.length <= 0 || lineTask.startsWith("## ")){
						break;
					}
					//如果没有冒号，表示未设置时间，默认设置planDefaultTaskTime
					if(lineTask.indexOf(":") == -1){
						lineTask = lineTask.replace("- [ ]","- [ ] " + planDefaultTaskTime);
					}
					//加入任务
					taskArray.push(lineTask.replace("- [ ] ","").replace(" ","|"));
					//加入原文到待修改
					modifyLabel.push(lineArray[i+j]);
				}
			}
		}

		let modifyContent = content;
		for(var i = 0;i < modifyLabel.length;i++){
			modifyContent = modifyContent.replace(modifyLabel[i] + "\n","");
			modifyContent = modifyContent.replace(modifyLabel[i],"");
		}
		//删除排期计划中这次生成任务的日期组
		if(planFileRemoveFlg){
			app.vault.modify(planFile,modifyContent);
		}
	}


	//---------汇总任务数据---------\\
	//任意日期，判断用不到，因为每日清单只对一天的时间处理（后面有个空格，用于Date格式识别）
	const placeholderDate = "2000-09-05 ";
	//根据时间对任务排序
	taskArray.sort(function(task1,task2) {
		let task1Time = new Date(placeholderDate + task1.split("|")[0]);
		let task2Time = new Date(placeholderDate + task2.split("|")[0]);
		return task1Time > task2Time ? 1 : -1;
	});
	//遍历不同的时间段，生成任务清单
	for(var i = 0;i < timeRegionArray.length;i++){
		const nowTimeRegion = timeRegionArray[i].split("|");
		const nowTimeRegionStartDate = new Date(placeholderDate + nowTimeRegion[0])
		const nowTimeRegionEndDate = new Date(placeholderDate + nowTimeRegion[1])
		//在任务清单中寻找当前时间段的任务
		let nowTimeRegionTaskArray = taskArray.filter(task => {
			let taskTime = new Date(placeholderDate + task.split("|")[0]);
			return (taskTime >= nowTimeRegionStartDate && taskTime <= nowTimeRegionEndDate);
		});
		if(nowTimeRegionTaskArray.length > 0){
			taskListStr += ("### " + nowTimeRegion[2] + "\n");
			for(var j = 0;j < nowTimeRegionTaskArray.length;j++){
				taskListStr += ("- [ ] " + nowTimeRegionTaskArray[j].replace("|"," ") + "\n");
			}
		}
	}
-%>
---
create: <% tp.file.creation_date () %>
tag: 日记,#todo/日常
---
<% taskListStr %>
```dataviewjs
//【今日工作】获取
//获取当天日报
const fileToday = new Date(dv.current().create);
const file = dv.page("002.工作/工作记录/日报/日报：" + fileToday.getFullYear() + "年" + (fileToday.getMonth() + 1) + "月" + fileToday.getDate() + "日");
if(file != null){
	dv.header("3","今日工作")
	dv.taskList(file.file.tasks,false)
}
```
```dataviewjs
//【今日文章】获取
//获取当天创建的文件
const fileToday = new Date(dv.current().create);
const todayFileArray = dv.pages().where(file => {
	const fileDate = new Date(file.file.cday);
	return (fileDate.getDate() == fileToday.getDate() && fileDate.getMonth() == fileToday.getMonth() && fileDate.getFullYear() && fileToday.getFullYear());
})

if(todayFileArray.length > 0){
	dv.header("3","今日文章")
}
for(var i = 0;i<todayFileArray.length;i++){
	dv.el("li","[[ " + todayFileArray[i].file.path + "|"+todayFileArray[i].file.name+" ]]\n");
}
```
### 结语
结束了{{DATE:yyyy年MMMD日}}。
````

````MARKDOWN TI:"周记模板"
<%*
//根据文件名获取当前年份和周
const fileName = tp.file.title;
const year = fileName.substring(fileName.indexOf("：") + 1,fileName.indexOf("年"));
//周计算问题，可能要？+1周
const week = parseInt(fileName.substring(fileName.indexOf("年") + 1,fileName.lastIndexOf("周"))) + 1;

// 此年1号是星期几
let firstDay = new Date(year,0,1).getDay()
// 方便计算，当为星期天时为7
if(firstDay == 0){
    firstDay = 7;
}

let oneFirstDay;
let oneLastDay;
// 如果1号刚好是星期一
if(firstDay == 1){
     oneFirstDay = year + '-01-01'
     oneLastDay = year + '-01-07'
} else {
    let jj = 8 - firstDay
    oneFirstDay = (year - 1) + '-12-' + (31 - firstDay + 2 > 9 ? 31 - firstDay + 2 : '0' + (31 - firstDay + 2))
    oneLastDay = year + '-01-' + (jj > 9 ? jj : '0' + jj)
}

let weekFirstDay;
let weekLastDay;
// 如果刚好是第一周
if(week == 1){
    weekFirstDay = oneFirstDay
    weekLastDay = oneLastDay
}else{
    weekFirstDay = addDate(oneLastDay,(week - 2) * 7 + 1)
    weekLastDay = addDate(oneLastDay,(week - 1) * 7)
}

//日期加减法  date参数为计算开始的日期，days为需要加的天数   
//格式:addDate('2017-1-11',20) 
function addDate(date,days){ 
    var d=new Date(date); 
    d.setDate(d.getDate()+days); 
    var m=d.getMonth()+1; 
    return d.getFullYear()+'-'+(m>9?m:'0'+m)+'-'+(d.getDate()>9?d.getDate():'0'+d.getDate()); 
}
-%>
---
weekFirstDay: <% weekFirstDay %>
weekLastDat: <% weekLastDay %>
create: <% tp.file.creation_date () %>
tag: 周记
---
```dataviewjs
const weekFirstDay = dv.current().weekFirstDay;
const weekLastDat = dv.current().weekLastDat;
const weekday=["星期日","星期一","星期二","星期三","星期四","星期五","星期六"];

//本周日程输出
let resultText = "";

//用于概要统计
let dailyCountComplete = 0;
let dailyCountFail = 0;
let workCountComplete = 0;
let workCountFail = 0;
let docCount = 0;
for(var i = 0;i <= 6; i++){
	const nowDate = new Date(weekFirstDay);
	nowDate.setDate(nowDate.getDate() + i);
	//格式化为日报日期
	const nowTime = nowDate.getFullYear() + "年"+ (nowDate.getMonth() + 1) + "月" + nowDate.getDate() + "日";

	//当天有日记才生成数据
	const dayFile = dv.page("003.自我/日记/日记："+nowTime+".md");
	if(dayFile != null){
		//标题
		resultText += "\n";
		resultText += "> [!TIP]- " +  weekday[nowDate.getDay()] +"\n> <span style='color: #72e9fd;font-weight:bold'>📗[[" + dayFile.file.path + "|"+dayFile.file.name+"]]</span>\n";
	
		//日程
		resultText += ">> [!SUCCESS] 日程表\n"
		if(dayFile != null && dayFile.file != null){
			const dayTask = dayFile.file.tasks;
			for(var rc = 0;rc < dayTask.length;rc ++){
				dayTask[rc].status == "x" ? dailyCountComplete++ : dailyCountFail++;
				resultText += ">> "+dayTask[rc].symbol+" ["+dayTask[rc].status+"] "+dayTask[rc].text+"\n";
			}
		}
		
		//工作
		resultText += " > "
		resultText += "\n >> [!example] 工作项\n"
		const dayWorkFile = dv.page("002.工作/工作记录/日报/日报："+nowTime+".md");
		if(dayWorkFile != null && dayWorkFile.file != null){
			const dayTask = dayWorkFile.file.tasks;
			for(var gz = 0;gz < dayTask.length;gz ++){
				dayTask[gz].status == "x" ? workCountComplete++ : workCountFail++;
				resultText += ">> "+dayTask[gz].symbol+" ["+dayTask[gz].status+"] "+dayTask[gz].text+"\n";
			}
		}
	
	
		//文章
		resultText += " >"
		resultText += "\n >> [!NOTE] 新文章\n";
		const dayNewDoc = dv.pages().where(file => {
			const fileDate = new Date(file.file.cday);
			return (fileDate.getDate() == nowDate.getDate() && fileDate.getMonth() == nowDate.getMonth() && fileDate.getFullYear() && nowDate.getFullYear());
	})
		for(var j = 0;j<dayNewDoc.length;j++){
			docCount++;
			resultText += ">> - [[ " + dayNewDoc[j].file.path + "|"+dayNewDoc[j].file.name+" ]]\n";
		}
	}
	
}

//输出-本周概要
dv.header(3,"本周概要")
let summaryResult = "> [!SUCCESS] 共完成 \n" +
"> - 🌞日程：<span style='color: #82eb82;font-weight:bold'>"+dailyCountComplete+"</span> \n" +
"> - 💼工作任务：<span style='color: #82eb82;font-weight:bold'>"+workCountComplete+"</span> \n" +
"> - ✒️新文章：<span style='color: #82eb82;font-weight:bold'>"+docCount+"</span> \n" +
"> \n" +
">> [!FAIL]- 未完成 \n" +
">> - 🌞日程：<span style='color: #f078a0;font-weight:bold'>"+dailyCountFail+"</span> \n" +
">> - 💼工作任务：<span style='color: #f078a0;font-weight:bold'>"+workCountFail+"</span> \n";
dv.span(summaryResult)
//输出-本周日程
dv.header(3,"本周日程")
dv.span(resultText)
//输出-补充内容
dv.header(3,"本周总结")
```
`````

### 配置插件
打开设置，从左下 *"第三方插件"* 列表找到插件名字，点击即可进入插件配置

#### **Templater**
- 配置模板文件夹
  ![image.png|670](https://cwl-img.oss-cn-beijing.aliyuncs.com/202305141626103.png)
- 配置日记和周记模板映射
  ![image.png|670](https://cwl-img.oss-cn-beijing.aliyuncs.com/202305141628623.png)

#### **Dataview**
- 打开JS支持选项
  ![image.png|670](https://cwl-img.oss-cn-beijing.aliyuncs.com/202305141643412.png)

#### *Calendar*
- 配置日历上显示周数，方便快速创建周记
  ![image.png|670](https://cwl-img.oss-cn-beijing.aliyuncs.com/202305141645301.png)
- 配置点击创建的周记信息
  ![image.png|670](https://cwl-img.oss-cn-beijing.aliyuncs.com/202305141647002.png)

#### *Day Planner*
- 设置模式为Commond mode
  ![image.png|670](https://cwl-img.oss-cn-beijing.aliyuncs.com/202305141641890.png)

#### *Checklist*
- 屏蔽模板文件夹
  ![image.png|670](https://cwl-img.oss-cn-beijing.aliyuncs.com/202305141649552.png)

#### ToggleList
- 配置任务切换类型
  ![image.png|670](https://cwl-img.oss-cn-beijing.aliyuncs.com/202305141651511.png)
- 配置任务切换快捷键
  ![image.png|670](https://cwl-img.oss-cn-beijing.aliyuncs.com/202305141652993.png)

- 点击下面的**HotKey**按钮，给这个操作设置快捷键
  ![image.png|670](https://cwl-img.oss-cn-beijing.aliyuncs.com/202305141655956.png)

#### Style Setting
- 放入CSS片段
  ![image.png|670](https://cwl-img.oss-cn-beijing.aliyuncs.com/202305141657489.png)
````CSS TI:"CSS片段"
/* @settings

name: GoblinCwl@css
id: GoblinCwl@css
settings:
    -   
        id: taskFailBoxColor
        title: 任务失败复选框颜色
        description: 更改任务状态为“/”时复选框的颜色
        type: variable-color
        opacity: false
        format: hex
        default: '#e66a51'
    -   
        id: taskFailTextColor
        title: 任务失败文本颜色
        description: 更改任务状态为“/”时文本的颜色
        type: variable-color
        opacity: false
        format: hex
        default: '#945b5b'
    - 
        id: dayPannel
        title: Day Pannel 日程表
        type: heading
        level: 1
        collapsed: true
    -   
        id: dayPannelTaskColor
        title: 日程表任务颜色
        description: 更改Day Pannel日程表中任务显示的颜色
        type: variable-color
        opacity: false
        format: hex
        default: '#FFFFFF'
    -   
        id: dayPannelHoverBackGroudColor
        title: 日程表高亮背景色
        description: 更改Day Pannel日程表高亮背景色
        type: variable-color
        opacity: false
        format: hex
        default: 'var(--interactive-accent-hover)'

*/

body {
    --dayPannelTaskColor: #FFFFFF;
    --dayPannelHoverBackGroudColor: var(--interactive-accent-hover);
    --taskFailBoxColor: #e66a51;
    --taskFailTextColor: #945b5b;
}

/*更改Day Pannel日程表中任务显示的颜色*/
.ei_Copy.svelte-e43ld1.svelte-e43ld1 {
    color: var(--dayPannelTaskColor)!important;
}

/**更改Day Pannel日程表hover背景色*/
.event_item.svelte-e43ld1.svelte-e43ld1:hover{
    background-color: var(--dayPannelHoverBackGroudColor)!important;
}

/**任务失败复选框颜色*/
.anp-custom-checkboxes [data-task="/"] > input[type=checkbox]:checked, .anp-custom-checkboxes [data-task="/"] > p > input[type=checkbox]:checked, .anp-custom-checkboxes [data-task="/"][type=checkbox]:checked {
    --checkbox-color: var(--taskFailBoxColor);
    --checkbox-color-hover: var(--taskFailBoxColor);
    border-color: var(--taskFailBoxColor) !important;
}

/*任务失败文本颜色*/
ul > li.task-list-item[data-task="/"], ul > li.task-list-item[data-task="/"] {
    text-decoration: var(--checklist-done-decoration);
    color: var(--taskFailTextColor);
}

/**日程时间线Track开关去掉多余的对勾*/
#scroll-controls.svelte-e43ld1 .toggle.svelte-e43ld1:after {
    background-color: unset;
}
````
- 在Style Setting的插件设置中可以找到这个CSS，可以配置颜色
  ![image.png|670](https://cwl-img.oss-cn-beijing.aliyuncs.com/202305141659647.png)
> 这里有显示BUG，同组颜色会显示一样，实际效果实现了就好
### 配置脚本
脚本中提供了非常简单的方式来自定义你的日程清单。
#### 时间段
```JAVASCRIPT TI:"时间段"
//时间段
const timeRegionArray = [
	"00:00|04:59|凌晨",
	"05:00|08:59|早晨",
	"09:00|11:29|上午",
	"11:30|13:29|中午",
	"13:30|17:59|下午",
	"18:00|23:59|晚上",
];
```
在脚本中找到这块代码，来定义日程中对任务的时间段分类，
格式为`时:分|时:分|时间段名称`，**时和分必须是两位数！**
此后，当日程生成时，会根据任务前面的时间来分类到对应的时间段。
如果想排序，更改代码中时间段的顺序就好。

#### 每日任务
```JAVASCRIPT TI:"每日固定任务"
//每日任务清单，预输入固定的每日任务
let taskArray = [
	"07:20|🌞起床",
	"07:25|🪥喝一杯水，洗漱",
	"07:30|🍵**泡茶**",
	"12:25|👀眼保健操",
	"12:30|💤午睡",
	"21:30|📚自考学习",
	"23:15|🥛热牛奶",
	"23:50|🌙洗漱，早睡",
];
```
在脚本中找到这块代码，来定义每天必定会创建的日程，
格式为`时:分|任务`，可以使用emoji和加粗/斜体，
此后，每一天的日程都会有上面的任务。

#### 每周任务
```JAVASCRIPT TI:"每周任务"
//周常任务清单
let weekTaskArray = [{},[],[],[],[],[],[],[]];
//想把任务加在周几，数字就填几
weekTaskArray[5].push("17:00|📰**周报**");
weekTaskArray[6].push("16:00|📞**给妈妈打电话**");
weekTaskArray[7].push("17:00|🧹*整理环境*");
weekTaskArray[7].push("22:00|🗒️**周记**");
```
在脚本中找到这块代码，来定义需要根据星期几来创建的任务，
在`weekTaskArray`后的中括号中输入要星期几创建任务，在后面输入任务格式，
例如，我需要在周四上午的10:30进行笔记整理任务，则另起一行输入`weekTaskArray[4].push("10:30|笔记整理");`
此后，在每周四的日程表中，会创建10:30分的笔记整理这个任务。

#### 每月任务
```JAVASCRIPT TI:"每月任务"
//月任务清单
let monthTaskArray = [{},[],[],[],[],[],[],[],[],[],[],[],[],[],[],[],[],[],[],[],[],[],[],[],[],[],[],[],[],[],[],[]];
//想把任务加在每月几号，数字就填几
monthTaskArray[10].push("17:00|🗒💴发工资咯~");
```
在脚本中找到这块代码，来定义需要每个月几号创建的任务，
在`monthTaskArray`后的中括号中输入要每月几号创建任务，在后面输入任务格式。
例如，我需要在每月30号19:00整理月账单，则另起一行输入`weekTaskArray[30].push("19:00|整理月账单");`
此后，在每月的30号的日程表中，会创建19:00整理月账单这个任务。
当然，此处只是根据每月几号，并不是每月最后一天，像2月可能就没有30号，一整个月都不会创建整理月账单，
如果有这方面需求，可以自己写js根据日期判断一下。

#### 排期计划任务
```JAVASCRIPT TI:"排期计划"
//排期计划文件
const planFile = tp.file.find_tfile("排期计划");
//生成日程后排期计划内当天的任务是否删除(true=删除，false=不删除)
const planFileRemoveFlg = true;
//排期计划中未指定精确时间的任务应该在哪个时间点
const planDefaultTaskTime = "09:00";
```
在脚本中找到这块代码，来定义不需要重复，只是未来的某一天需要创建的日程。
首先需要有一个 **"排期计划"** 的笔记，当然你也可以自己明明，修改代码中对应的名字就行
在该文件中以如下形式写下未来日程需要创建的任务：
```MARKDOWN
## 2023年5月17日
- [ ] 13:30 🐱寄养小猫
```
此后，在2023年5月17日这一天，就会创建13:30的🐱寄养小猫任务。
如果你希望创建后保留排期计划中设置的任务，则将`planFileRemoveFlg`的值改为`false`，
否则，在当天创建完日程后，排期计划的笔记中将会自动删除刚刚创建日程时创建的任务。
此外，当你在排期计划中没有明确指定时间点时，将会默认以"09:00"的时间创建任务，
当然，你也可以更改`planDefaultTaskTime`的值来更改默认的时间点。
**planDefaultTaskTime的值中时钟和分钟必须是两位数。**

#### 动态任务
```JAVASCRIPT TI:"动态任务"
//动态任务，需要独立判断
//昨天日期
let yesterdayDate = new Date(today);
yesterdayDate.setDate(today.getDate() - 1);
// 昨天日记文件名
let fullFileName = "日记："+yesterdayDate.getFullYear()+"年"+(yesterdayDate.getMonth()+1)+"月"+yesterdayDate.getDate()+"日";
const file = tp.file.find_tfile(fullFileName);
if(file != null){
	const content = await app.vault.cachedRead(file);
	const contentStr = content.toString();
	// 隔一天跑步散步
	if(contentStr.contains("- [x] 20:30 👟跑步")){
		taskArray.push("20:30|👞散步");
	}else{
		taskArray.push("20:30|👟跑步");
	}
}
```
可能你还需要一些更有逻辑的任务创建，但是这需要一些Javascript基础来编写。
如代码所示，此处编写的是如果昨天的日程中我完成了跑步任务，那么今天就生成散步任务，否则生成跑步任务。

### 今日工作和今日文章
```` MARKDOWN
```dataviewjs
//【今日工作】获取
//获取当天日报
const fileToday = new Date(dv.current().create);
const file = dv.page("002.工作/工作记录/日报/日报：" + fileToday.getFullYear() + "年" + (fileToday.getMonth() + 1) + "月" + fileToday.getDate() + "日");
if(file != null){
	dv.header("3","今日工作")
	dv.taskList(file.file.tasks,false)
}
```
```dataviewjs
//【今日文章】获取
//获取当天创建的文件
const fileToday = new Date(dv.current().create);
const todayFileArray = dv.pages().where(file => {
	const fileDate = new Date(file.file.cday);
	return (fileDate.getDate() == fileToday.getDate() && fileDate.getMonth() == fileToday.getMonth() && fileDate.getFullYear() && fileToday.getFullYear());
})

if(todayFileArray.length > 0){
	dv.header("3","今日文章")
}
for(var i = 0;i<todayFileArray.length;i++){
	dv.el("li","[[ " + todayFileArray[i].file.path + "|"+todayFileArray[i].file.name+" ]]\n");
}
```
`````
你肯定注意到模板下面还有这些内容，这些内容是个人日程外对日记的一个附加信息。
比如可以从当天的日报中获取今天日报的任务，显示在日记中，
> 任务：指"- [ ]"开头的东西，不管你日报是什么样式，只是读取日报中的任务

以及查询今天创建的文章，显示在日记中。
今日文章的生成可以直接使用。
今日工作则需要你有一个日报文件夹，并且在代码中设置对应的路劲和名称，
例如模板中的路劲和名称就是 *"002.工作/工作记录/日报/日报：XXXX年X月X日"*
![image.png|250](https://cwl-img.oss-cn-beijing.aliyuncs.com/202305141727582.png)

### 周记
直接从日历中点击周数创建周记即可，你也可以通过"QuickAdd"创建周记指令来生成周记。
周记中也有我关于工作日报的配置，不需要工作日报可以删除对应代码块。

## 总结
个人使用中这套模板还是非常方便的，也期待发现更方便的方式管理日程。

## 备注
1. 创建日记后，记得将日记关联到Day Pannel
   ![image.png|670](https://cwl-img.oss-cn-beijing.aliyuncs.com/202305141816852.png)
