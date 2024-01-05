- [日志格式](#日志格式)
- [最佳实践](#最佳实践)


## 日志格式

[https://betterstack.com/community/guides/logging/log-formatting/](https://betterstack.com/community/guides/logging/log-formatting/)

关注点：

1. 每条日志记录清晰并保持一致
2. 日志编码方式
3. 组织日志的上下文字段
4. 定义如何分隔日志记录

常见格式类型：

1. 非结构化日志

   ```
   // 没有一致性结构，难以排查问题
   // 可用sed/awk等命令过滤，但只是临时解决方案
   
   1687927940843: Starting application on port 3000
   Encountered an unexpected error while backing up the database
   [2023-06-28T04:49:48.113Z]: Device XYZ123 went offline
   ```

   

2. 半结构化日志

   ```
   // 有大致结构，时间戳/日志级别/pid/tid/appname
   // 但缺少可识别格式，message嵌入了所有其他上下文细节
   
   2023-06-28 19:09:48.801818 I [969609:60] MyApp -- Starting application on port 3000
   2023-06-28 19:09:48.801844 I [969609:60] MyApp -- Device XYZ123 went offline
   2023-06-28 19:09:48.801851 E [969609:60 main.rb:13] MyApp -- Service "ABCDE" stopped unexpectedly. Restarting the service
   ```

   

3. 结构化日志

   ```
   // 所有日志条目都遵从可识别的一致性格式，但降低了可读性
   // 便于自动化管理工具搜索/分析/监控
   
   {
     "host": "fedora",
     "application": "Semantic Logger",
     "timestamp": "2023-06-28T17:20:19.409882Z",
     "level": "info",
     "level_index": 2,
     "pid": 982617,
     "thread": "60",
     "name": "MyApp",
     "message": "Starting application on port 3000"
   }
   ```

   **可通过配置日志格式提高可读性，默认json格式！**





## 最佳实践

1. 使用JSON格式

   可以是本地自己实现或者使用外部插件.

   如Go：

   ```
   package main
   
   import (
       "log/slog"
       "os"
   )
   
   func main() {
       logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
       logger.Info("an info message")
   }
   // {"time":"2023-09-14T11:49:17.671587229+02:00","level":"INFO","msg":"an info message"}
   ```

   如Nginx：

   ```
   ...
   http {
     log_format custom_json escape=json
     '{'
       '"timestamp":"$time_iso8601",'
       '"pid":"$pid",'
       '"remote_addr":"$remote_addr",'
       '"remote_user":"$remote_user",'
       '"request":"$request",'
       '"status": "$status",'
       '"body_bytes_sent":"$body_bytes_sent",'
       '"request_time_ms":"$request_time",'
       '"http_referrer":"$http_referer",'
       '"http_user_agent":"$http_user_agent"'
     '}';
     access_log /var/log/nginx/access.json custom_json;
   }
   // 127.0.0.1 alice Alice [07/May/2021:10:44:53 +0200] "GET / HTTP/1.1" 200 396 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/90.0.4531.93 Safari/537.36"
   ```

   

2. 基于字符串的标准化日志级别

   日志级别通常用int标识

   如Go：

   ```
   {
     debug: -4,
     info: 0,
     warn: 4,
     error: 8,
   }
   ```

   如Node的Pino框架：

   ```
   {
     trace: 10,
     debug: 20,
     info: 30,
     warn: 40,
     error: 50,
     fatal: 60,
   }
   ```

   但输出以字符串标识：

   ```
   {"level":"INFO","time":"2023-03-15T13:07:39.105777557+01:00","msg":"Info message"}
   ```



3. 推荐以ISO-8601记录时间戳

   ```
   // 可读性好，可精确到纳秒，可选时区信息
   
   2023-09-10T12:34:56.123456789Z
   2023-09-10T12:34:56+02:00
   2023-11-17T12:34:56.123456789+01:00
   ```



4. 包含详细的日志源信息

   ```
   // 以便快速精准定位问题
   
   {
     "time": "2023-05-24T19:39:27.005Z",
     "level": "DEBUG",
     "source": {
       "function": "main.main",
       "file": "app/main.go",
       "line": 30
     },
     "msg": "Debug message"
   }
   ```

   

5. 包含构建version/提交hash

   ```
   // 以确定当前的代码状态
   
   {
     "time": "2023-06-29T06:37:38.429463999+02:00",
     "level": "ERROR",
     "source": {
       "function": "main.main",
       "file": "app/main.go",
       "line": 38
     },
     "msg": "an unexpected error",
     "build_info": {
       "go_version": "go1.20.2",
       "commit_hash": "9b0695e1c4732a2ea2c8ac678472c4c3c235101b"
     }
   }
   ```



6. 错误日志中包含堆栈信息

   设计良好的框架能自动捕获异常的堆栈信息。

   如Python标准日志库：

   ```
   {
     "name": "__main__",
     "timestamp": "2023-02-06T09:28:54Z",
     "severity": "ERROR",
     "filename": "example.py",
     "lineno": 26,
     "process": 913135,
     "message": "division by zero",
     "exc_info": "Traceback (most recent call last):\n  File \"/home/betterstack/community/python-logging/example.py\", line 24, in <module>\n    1 / 0\n    ~~^~~\nZeroDivisionError: division by zero"
   }
   ```

   堆栈信息也可以格式化：

   ```
   {
     "event": "Cannot divide one by zero!",
     "level": "error",
     "timestamp": "2023-07-31T07:00:31.526266Z",
     "exception": [
       {
         "exc_type": "ZeroDivisionError",
         "exc_value": "division by zero",
         "syntax_error": null,
         "is_cause": false,
         "frames": [
           {
             "filename": "/home/betterstack/structlog_demo/app.py",
             "lineno": 16,
             "name": "<module>",
             "line": "",
             "locals": {
               "__name__": "__main__",
               "__doc__": "None",
               "__package__": "None",
               "__loader__": "<_frozen_importlib_external.SourceFileLoader object at 0xffffaa2f3410>",
               "__spec__": "None",
               "__annotations__": "{}",
               "__builtins__": "<module 'builtins' (built-in)>",
               "__file__": "/home/betterstack/structlog_demo/app.py",
               "__cached__": "None",
               "structlog": "\"<module 'structlog' from '/home/betterstack/structlog_demo/venv/lib/python3.11/site-\"+32",
               "logger": "'<BoundLoggerLazyProxy(logger=None, wrapper_class=None, processors=None, context_'+55"
             }
           }
         ]
       }
     ]
   }
   ```

   

7. 标准化上下文字段

   日志信息通常会嵌入上下文信息，以记录抛出日志的一些关键信息

   ```
   slog.Info("User logged in", slog.Int("user_id", 42))
   
   {
     "time": "2023-11-16T08:58:20.474534594+02:00",
     "level": "INFO",
     "msg": "User logged in",
     "user_id": 42
   }
   ```

   

8. 使用ID标识关联的请求日志

   如在一个HTTP请求的各个生命周期都可能生成日志，需要一个ID来标识同一请求的所有日志

   ```
   {
     "timestamp": "2023-09-10T15:30:45.123456Z",
     "correlation_id": "9ea8f2b4-639e-4de7-b406-f6cd3a155e9f",
     "level": "INFO",
     "message": "Received incoming HTTP request",
     "request": {
       "method": "GET",
       "path": "/api/resource",
       "remote_address": "192.168.1.100"
     }
   }
   ```



9. 实现对象的选择性日志

   以过滤敏感信息，只打印必要字段

   ```
   type User struct {
       ID    string `json:"id"`
       Name  string `json:"name"`
       Email string `json:"email"`
       Password string `json:"password"`
   }
   
   func (u *User) LogValue() slog.Value {
       return slog.StringValue(u.ID)
   }
   ```

   