---
layout: post
title:  "【Tips】iOS日期操作那些事"
date:   2017-05-03
excerpt: "最近开发过程中，业务中一个预约的模块，用到了很多关于时间的操作，所以今天借此机会总结下开发中会常用到的日期以及时间的操作。"
tag:
- Swift
comments: true
---

### 前言

最近开发过程中，业务中一个预约的模块，用到了很多关于时间的操作，所以今天借此机会总结下开发中会常用到的日期以及时间的操作。

> 总的来说，不论什么操作，合理运用`DateFormatter`、`Calendar`和`DateComponents`就可以解决，同时还会省下很多时间。

#### 获取某个月的天数
```
let date = Date()
let range = calendar.range(of: .day, in: .month, for: date)
let count = range.count  
```

#### 获取当前日期字符串
```
let date = Date()
let dateFormatter = DateFormatter()
dateFormatter.dateFormat = "yyyy-MM-dd HH:mm:ss"
dateFormatter.timeZone = TimeZone(secondsFromGMT: 8)
let currentDateStr = dateFormatter.string(from: date)
```
输出的日期格式可能是多种多样的，只需要将常见的日期元素说明符进行搭配就好

- EEEE：“星期”的全名（比如Monday）。（EEE代表Mon）
- MMMM：“月份”的全名（比如October）。（MMM代表Oct, MM代表数字形式的月，例如05或11）
- dd：某月的第几天（例如，09或15）
- yyyy：四位字符串表示“年”（例如2015）
- HH：两位字符串表示“小时”（例如08或19）
- mm：两位字符串表示“分钟”（例如05或54）
- ss：两位字符串表示“秒”
- zzz：三位字符串表示“时区”（例如GMT）
- GGG：公元前BC或公元后AD

> 详细的日期格式请戳[这里](http://unicode.org/reports/tr35/tr35-6.html#Date_Format_Patterns)

#### 根据字符串获取Date类型结构体
```
let dateFormatter = DateFormatter()
dateFormatter.dateFormat = "yyyy-MM-dd HH:mm:ss"
let dateStr = "2016-05-07 17:50:00" //注意，日期字符串的格式必须和dateFormatter的格式一致，否则转成日期就会为nil
let currentDate = dateFormatter.date(from: dateStr)
```

#### 计算两个日期的差值
```
let date = Date()

let dateFormatter = DateFormatter()
dateFormatter.dateFormat = "yyyy-MM-dd HH:mm:ss"
let dateStr = "2016-05-07 17:50:00" //注意，日期字符串的格式必须和dateFormatter的格式一致，否则转成日期就会为nil
let currentDate = dateFormatter.date(from: dateStr)

let calendar = Calendar.current
let component: Set<Calendar.Component> = [.year, .month, .day, .hour, .minute, .second]
let dateComponent = calendar.dateComponents(component, from: currentDate!, to: date)
```

#### 根据某一日期计算后面的连续多少天(比如当前开始的一周)
```
let date = Date()
var datesOfOneWeek = [Date]()
for i in 1...7 {
    datesOfOneWeek.append(date.addingTimeInterval((Double)(i) * 24 * 60 * 60))
}
```

#### 判断时间是否在某个时间段内（勿扰模式）
```
    /// 将给定的某个小时，转成当天的日期格式
    ///
    /// - Parameter hour: 24小时制的某个小时
    /// - Returns: 该小时的Date类型数据
    func getCustomDateWithHour(_ hour: NSInteger) -> Date {
        let currentDate = Date()
        let currentCalendar = Calendar.current
        
        let currentComponent = currentCalendar.dateComponents([.year, .month, .day, .hour, .minute, .second], from: currentDate)
        
        var resultComponent = DateComponents()
        resultComponent.setValue(currentComponent.year, for: .year)
        resultComponent.setValue(currentComponent.month, for: .month)
        resultComponent.setValue(currentComponent.day, for: .day)
        resultComponent.setValue(hour, for: .hour)
        
        return currentCalendar.date(from: resultComponent)!
    }

    /// 比较当前时间是否在当天[fromhour,toHour] 这个区间
    func isBetweenFromHour(_ fromHour: NSInteger, toHour: NSInteger) -> Bool {
        let date1 = self.getCustomDateWithHour(fromHour)
        let date2 = self.getCustomDateWithHour(toHour)
        let currentDate = Date()
        
        if (currentDate.compare(date1) == ComparisonResult.orderedDescending && currentDate.compare(date2) == ComparisonResult.orderedAscending){
            return true
        } else {
            return false
        }
    }
```

### 最后
想到再加😁
