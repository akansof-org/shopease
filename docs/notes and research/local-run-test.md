
# Run applications locally

Purpose: To find out how to build the application. how to run it, what env vars it needs and other services or third party softwares/resources it depends on.

This helps to write the dockerfile that will package the app as a container.k

## Product Catalog Service

Works fine

```Shell
  productcatalogservice git:(main) ✗ ./productcatalogservice
{"message":"Tracing disabled.","severity":"info","timestamp":"2026-08-03T11:05:59.11902Z"}
{"message":"Profiling enabled.","severity":"info","timestamp":"2026-08-03T11:05:59.119412Z"}
{"message":"starting grpc server at :3550","severity":"info","timestamp":"2026-08-03T11:05:59.120495Z"}
{"message":"loading catalog from local products.json file...","severity":"info","timestamp":"2026-08-03T11:05:59.122486Z"}
{"message":"successfully parsed product catalog json","severity":"info","timestamp":"2026-08-03T11:05:59.123843Z"}
{"message":"failed to start profiler: project ID must be specified in the configuration if running outside of GCP","severity":"warning","timestamp":"2026-08-03T11:05:59.125523Z"}
{"message":"sleeping 10s to retry initializing Stackdriver profiler","severity":"info","timestamp":"2026-08-03T11:05:59.126481Z"}
{"message":"failed to start profiler: project ID must be specified in the configuration if running outside of GCP","severity":"warning","timestamp":"2026-08-03T11:06:09.127679Z"}
{"message":"sleeping 20s to retry initializing Stackdriver profiler","severity":"info","timestamp":"2026-08-03T11:06:09.127776Z"}
{"message":"failed to start profiler: project ID must be specified in the configuration if running outside of GCP","severity":"warning","timestamp":"2026-08-03T11:06:29.129037Z"}
{"message":"sleeping 30s to retry initializing Stackdriver profiler","severity":"info","timestamp":"2026-08-03T11:06:29.12913Z"}
{"message":"could not initialize Stackdriver profiler after retrying, giving up","severity":"warning","timestamp":"2026-08-03T11:06:59.133466Z"}
```

## Recommendation Service

Ran into:

```Shell
➜  recommendationservice git:(main) ✗ python recommendation_server.py
{"timestamp": 1785755056.9402308, "severity": "INFO", "name": "recommendationservice-server", "message": "initializing recommendationservice"}
{"timestamp": 1785755056.940313, "severity": "INFO", "name": "recommendationservice-server", "message": "Profiler enabled."}
{"timestamp": 1785755056.941677, "severity": "INFO", "name": "recommendationservice-server", "message": "Tracing disabled."}
Traceback (most recent call last):
  File "/Users/aankansah/Code/akansof/10-downloads/shopease/src/recommendationservice/recommendation_server.py", line 133, in <module>
    raise Exception('PRODUCT_CATALOG_SERVICE_ADDR environment variable not set')
Exception: PRODUCT_CATALOG_SERVICE_ADDR environment variable not set
```

I needed to set:

```Shell
➜  recommendationservice git:(main) ✗ export PRODUCT_CATALOG_SERVICE_ADDR=localhost:3550
```

Now it works:

```Shell
➜  recommendationservice git:(main) ✗ python recommendation_server.py
{"timestamp": 1785755171.158591, "severity": "INFO", "name": "recommendationservice-server", "message": "initializing recommendationservice"}
{"timestamp": 1785755171.1586869, "severity": "INFO", "name": "recommendationservice-server", "message": "Profiler enabled."}
{"timestamp": 1785755171.160419, "severity": "INFO", "name": "recommendationservice-server", "message": "Tracing disabled."}
{"timestamp": 1785755171.160476, "severity": "INFO", "name": "recommendationservice-server", "message": "product catalog address: localhost:3550"}
{"timestamp": 1785755171.176817, "severity": "INFO", "name": "recommendationservice-server", "message": "listening on port: 8080"}
```

## Shipping Service

```Shell
# Build the application

➜  shippingservice git:(main) ✗ go build -o shippingservice .

# start the installed binary

➜  shippingservice git:(main) ✗ ./shippingservice
{"message":"Tracing enabled, but temporarily unavailable","severity":"info","timestamp":"2026-08-03T19:29:08.036203Z"}
{"message":"See https://github.com/GoogleCloudPlatform/microservices-demo/issues/422 for more info.","severity":"info","timestamp":"2026-08-03T19:29:08.036822Z"}
{"message":"Profiling enabled.","severity":"info","timestamp":"2026-08-03T19:29:08.037218Z"}
{"message":"Stats enabled, but temporarily unavailable","severity":"info","timestamp":"2026-08-03T19:29:08.03805Z"}
{"message":"Shipping Service listening on port :50051","severity":"info","timestamp":"2026-08-03T19:29:08.038294Z"}
{"message":"failed to start profiler: project ID must be specified in the configuration if running outside of GCP","severity":"warning","timestamp":"2026-08-03T19:29:08.04165Z"}
{"message":"sleeping 10s to retry initializing Stackdriver profiler","severity":"info","timestamp":"2026-08-03T19:29:08.041746Z"}
{"message":"failed to start profiler: project ID must be specified in the configuration if running outside of GCP","severity":"warning","timestamp":"2026-08-03T19:29:18.042635Z"}
{"message":"sleeping 20s to retry initializing Stackdriver profiler","severity":"info","timestamp":"2026-08-03T19:29:18.042712Z"}
{"message":"failed to start profiler: project ID must be specified in the configuration if running outside of GCP","severity":"warning","timestamp":"2026-08-03T19:29:38.043808Z"}
{"message":"sleeping 30s to retry initializing Stackdriver profiler","severity":"info","timestamp":"2026-08-03T19:29:38.043855Z"}
```

## Cart Service

```Shell
# Publish the project with dotnet

➜  cartservice git:(main) ✗ dotnet publish -c release -o ./publish<200b>
Restore complete (16.1s)
  cartservice net10.0 succeeded (5.8s) → publish/
  cartservice.tests net10.0 succeeded (0.6s) → publish/
  cartservice.sln succeeded with 1 warning(s) (0.0s)
    /usr/local/share/dotnet/sdk/10.0.302/Current/SolutionFile/ImportAfter/Microsoft.NET.Sdk.Solution.targets(36,5): warning NETSDK1194: The "--output" option isn't supported when building a solution. Specifying a solution-level output path results in all projects copying outputs to the same directory, which can lead to inconsistent builds.

Build succeeded with 1 warning(s) in 22.7s


#############################
# Attempted Other Ways to run the App
#############################

# 1. Find the .csproj file and run it

➜  cartservice git:(main) ✗ find src -name "*.csproj"
src/cartservice.csproj

➜  cartservice git:(main) ✗ dotnet run --project src/cartservice.csproj
Redis cache host(hostname+port) was not specified. Starting a cart service using in memory store
info: Microsoft.Hosting.Lifetime[14]
      Now listening on: http://localhost:5000
info: Microsoft.Hosting.Lifetime[0]
      Application started. Press Ctrl+C to shut down.
info: Microsoft.Hosting.Lifetime[0]
      Hosting environment: Production
info: Microsoft.Hosting.Lifetime[0]
      Content root path: /Users/aankansah/Code/akansof/10-downloads/shopease/src/cartservice/src
^Cinfo: Microsoft.Hosting.Lifetime[0]
      Application is shutting down...


# 2. cd into the src directory and run the app with dotnet run

➜  src git:(main) ✗ dotnet run
Redis cache host(hostname+port) was not specified. Starting a cart service using in memory store
info: Microsoft.Hosting.Lifetime[14]
      Now listening on: http://localhost:5000
info: Microsoft.Hosting.Lifetime[0]
      Application started. Press Ctrl+C to shut down.
info: Microsoft.Hosting.Lifetime[0]
      Hosting environment: Production
info: Microsoft.Hosting.Lifetime[0]
      Content root path: /Users/aankansah/Code/akansof/10-downloads/shopease/src/cartservice/src
^Cinfo: Microsoft.Hosting.Lifetime[0]
      Application is shutting down...
```

## Currency Service

```Shell
# install project first with `npm install` then start the server

➜  currencyservice git:(main) ✗ node server.js
{"severity":"info","time":1785784368977,"pid":45009,"hostname":"Augustines-MacBook-Pro.local","name":"currencyservice-server","message":"Profiler enabled."}
{"severity":"info","time":1785784369463,"pid":45009,"hostname":"Augustines-MacBook-Pro.local","name":"currencyservice-server","message":"Tracing disabled."}
{"severity":"info","time":1785784369476,"pid":45009,"hostname":"Augustines-MacBook-Pro.local","name":"currencyservice-server","message":"Starting gRPC server on port 7000..."}
E No address added out of total 1 resolved
(node:45009) DeprecationWarning: Calling start() is no longer necessary. It can be safely omitted.
(Use `node --trace-deprecation ...` to show where the warning was created)
{"severity":"info","time":1785784369531,"pid":45009,"hostname":"Augustines-MacBook-Pro.local","name":"currencyservice-server","message":"CurrencyService gRPC server started on port 7000"}
/Users/aankansah/Code/akansof/10-downloads/shopease/src/currencyservice/node_modules/@grpc/grpc-js/build/src/server.js:853
                    throw new Error('server must be bound in order to start');
                          ^

Error: server must be bound in order to start
    at Server.start (/Users/aankansah/Code/akansof/10-downloads/shopease/src/currencyservice/node_modules/@grpc/grpc-js/build/src/server.js:853:27)
    at Server.deprecated (node:internal/util:167:12)
    at /Users/aankansah/Code/akansof/10-downloads/shopease/src/currencyservice/server.js:193:14
    at /Users/aankansah/Code/akansof/10-downloads/shopease/src/currencyservice/node_modules/@grpc/grpc-js/build/src/server.js:616:25
    at process.processTicksAndRejections (node:internal/process/task_queues:95:5)

Node.js v20.20.2
```

## Payment Service

```Shell
➜  paymentservice git:(main) ✗ ls
charge.js         genproto.sh       logger.js         package-lock.json proto
Dockerfile        index.js          node_modules      package.json      server.js
➜  paymentservice git:(main) ✗ node index.js
{"severity":"info","time":1785784674583,"pid":47749,"hostname":"Augustines-MacBook-Pro.local","name":"paymentservice-server","message":"Profiler enabled."}
{"severity":"info","time":1785784675921,"pid":47749,"hostname":"Augustines-MacBook-Pro.local","name":"paymentservice-server","message":"Tracing disabled."}
(node:47749) DeprecationWarning: Calling start() is no longer necessary. It can be safely omitted.
(Use `node --trace-deprecation ...` to show where the warning was created)
{"severity":"info","time":1785784676009,"pid":47749,"hostname":"Augustines-MacBook-Pro.local","name":"paymentservice-server","message":"PaymentService gRPC server started on port undefined"}
/Users/aankansah/Code/akansof/10-downloads/shopease/src/paymentservice/node_modules/@grpc/grpc-js/build/src/server.js:853
                    throw new Error('server must be bound in order to start');
                          ^

Error: server must be bound in order to start
    at Server.start (/Users/aankansah/Code/akansof/10-downloads/shopease/src/paymentservice/node_modules/@grpc/grpc-js/build/src/server.js:853:27)
    at Server.deprecated (node:internal/util:167:12)
    at /Users/aankansah/Code/akansof/10-downloads/shopease/src/paymentservice/server.js:65:16
    at /Users/aankansah/Code/akansof/10-downloads/shopease/src/paymentservice/node_modules/@grpc/grpc-js/build/src/server.js:616:25

Node.js v20.20.2
➜  paymentservice git:(main) ✗
```

## Checkout Service

```Shell
# Check for gradlew in the project and make it executable

➜  adservice git:(main) ✗ ls
build           Dockerfile      gradle          gradlew.bat     settings.gradle
build.gradle    genproto.sh     gradlew         README.md       src
➜  adservice git:(main) ✗ chmod +x gradlew
➜  adservice git:(main) ✗ ./gradlew build


# Install the project
➜  adservice git:(main) ✗ ./gradlew installDist

[Incubating] Problems report is available at: file:///Users/aankansah/Code/akansof/10-downloads/shopease/src/adservice/build/reports/problems/problems-report.html

Deprecated Gradle features were used in this build, making it incompatible with Gradle 9.0.

You can use '--warning-mode all' to show the individual deprecation warnings and determine if they come from your own scripts or plugins.

For more on this, please refer to https://docs.gradle.org/8.14.5/userguide/command_line_interface.html#sec:command_line_warnings in the Gradle documentation.

BUILD SUCCESSFUL in 826ms
10 actionable tasks: 1 executed, 9 up-to-date


# Check for the executable build file

➜  adservice git:(main) ✗ find build/install -type f -perm -111 -print
build/install/hipstershop/bin/AdService
build/install/hipstershop/bin/AdServiceClient.bat
build/install/hipstershop/bin/jar/MANIFEST.MF
build/install/hipstershop/bin/compileJava/previous-compilation-data.bin
build/install/hipstershop/bin/AdServiceClient
build/install/hipstershop/bin/AdService.bat

# Start the app
➜  adservice git:(main) ✗ PORT=9555 build/install/hipstershop/bin/AdService
{"instant":{"epochSecond":1785786338,"nanoOfSecond":731774000},"thread":"main","level":"INFO","loggerName":"hipstershop.AdService","message":"AdService starting.","endOfBatch":false,"loggerFqcn":"org.apache.logging.log4j.spi.AbstractLogger","threadId":1,"threadPriority":5,"logging.googleapis.com/trace":"${ctx:traceId}","logging.googleapis.com/spanId":"${ctx:spanId}","logging.googleapis.com/traceSampled":"${ctx:traceSampled}","time":"2026-08-03T19:45:38.731Z"}
{"instant":{"epochSecond":1785786338,"nanoOfSecond":731776000},"thread":"Thread-0","level":"INFO","loggerName":"hipstershop.AdService","message":"Stats enabled, but temporarily unavailable","endOfBatch":false,"loggerFqcn":"org.apache.logging.log4j.spi.AbstractLogger","threadId":30,"threadPriority":5,"logging.googleapis.com/trace":"${ctx:traceId}","logging.googleapis.com/spanId":"${ctx:spanId}","logging.googleapis.com/traceSampled":"${ctx:traceSampled}","time":"2026-08-03T19:45:38.731Z"}
{"instant":{"epochSecond":1785786338,"nanoOfSecond":779281000},"thread":"Thread-0","level":"INFO","loggerName":"hipstershop.AdService","message":"Tracing enabled but temporarily unavailable","endOfBatch":false,"loggerFqcn":"org.apache.logging.log4j.spi.AbstractLogger","threadId":30,"threadPriority":5,"logging.googleapis.com/trace":"${ctx:traceId}","logging.googleapis.com/spanId":"${ctx:spanId}","logging.googleapis.com/traceSampled":"${ctx:traceSampled}","time":"2026-08-03T19:45:38.779Z"}
{"instant":{"epochSecond":1785786338,"nanoOfSecond":780163000},"thread":"Thread-0","level":"INFO","loggerName":"hipstershop.AdService","message":"See https://github.com/GoogleCloudPlatform/microservices-demo/issues/422 for more info.","endOfBatch":false,"loggerFqcn":"org.apache.logging.log4j.spi.AbstractLogger","threadId":30,"threadPriority":5,"logging.googleapis.com/trace":"${ctx:traceId}","logging.googleapis.com/spanId":"${ctx:spanId}","logging.googleapis.com/traceSampled":"${ctx:traceSampled}","time":"2026-08-03T19:45:38.780Z"}
{"instant":{"epochSecond":1785786338,"nanoOfSecond":780590000},"thread":"Thread-0","level":"INFO","loggerName":"hipstershop.AdService","message":"Tracing enabled - Stackdriver exporter initialized.","endOfBatch":false,"loggerFqcn":"org.apache.logging.log4j.spi.AbstractLogger","threadId":30,"threadPriority":5,"logging.googleapis.com/trace":"${ctx:traceId}","logging.googleapis.com/spanId":"${ctx:spanId}","logging.googleapis.com/traceSampled":"${ctx:traceSampled}","time":"2026-08-03T19:45:38.780Z"}
{"instant":{"epochSecond":1785786338,"nanoOfSecond":946253000},"thread":"main","level":"INFO","loggerName":"hipstershop.AdService","message":"Ad Service started, listening on 9555","endOfBatch":false,"loggerFqcn":"org.apache.logging.log4j.spi.AbstractLogger","threadId":1,"threadPriority":5,"logging.googleapis.com/trace":"${ctx:traceId}","logging.googleapis.com/spanId":"${ctx:spanId}","logging.googleapis.com/traceSampled":"${ctx:traceSampled}","time":"2026-08-03T19:45:38.946Z"}
^C*** shutting down gRPC ads server since JVM is shutting down
*** server shut down
```

## Ad Service

```Shell
# Install Ad Service

➜  adservice git:(main) ✗ ./gradlew installDist

[Incubating] Problems report is available at: file:///Users/aankansah/Code/akansof/10-downloads/shopease/src/adservice/build/reports/problems/problems-report.html

Deprecated Gradle features were used in this build, making it incompatible with Gradle 9.0.

You can use '--warning-mode all' to show the individual deprecation warnings and determine if they come from your own scripts or plugins.

For more on this, please refer to https://docs.gradle.org/8.14.5/userguide/command_line_interface.html#sec:command_line_warnings in the Gradle documentation.

BUILD SUCCESSFUL in 826ms
10 actionable tasks: 1 executed, 9 up-to-date
```

## Email Service

```Shell
# Install the email service dependencies
➜  emailservice git:(main) ✗ python3 -m pip install -r requirements.txt
Requirement already satisfied: cachetools==5.3.2 in /Library/Frameworks/Python.framework/Versions/3.14/lib/python3.14/site-packages (from -r requirements.txt (line 3)) (5.3.2)
Requirement already satisfied: certifi==2024.7.4 in /Library/Frameworks/Python.framework/Versions/3.14/lib/python3.14/site-packages (from -r requirements.txt (line 5)) (2024.7.4)
Requirement already satisfied: charset-normalizer==3.3.2 in /Library/Frameworks/Python.framework/Versions/3.14/lib/python3.14/site-packages (from -r requirements.txt (line 7)) (3.3.2)
Requirement already satisfied: google-api-core==2.28.1 in /Library/Frameworks/Python.framework/Versions/3.14/lib/python3.14/site-packages (from google-api-core[grpc]==2.28.1->-r requirements.txt (line 9)) (2.28.1)
Requirement already satisfied: google-auth==2.23.4 in /Library/Frameworks/Python.framework/Versions/3.14/lib/python3.14/site-packages (from -r requirements.txt (line 13)) (2.23.4)
Collecting google-cloud-trace==1.17.0 (from -r requirements.txt (line 17))
  Downloading google_cloud_trace-1.17.0-py3-none-any.whl.metadata (9.8 kB)
Requirement already satisfied: googleapis-common-protos==1.72.0 in /Library/Frameworks/Python.framework/Versions/3.14/lib/python3.14/site-packages (from -r requirements.txt (line 19)) (1.72.0)
Requirement already satisfied: grpcio==1.76.0 in /Library/Frameworks/Python.framework/Versions/3.14/lib/python3.14/site-packages (from -r requirements.txt (line 24)) (1.76.0)
Requirement already satisfied: grpcio-health-checking==1.76.0 in /Library/Frameworks/Python.framework/Versions/3.14/lib/python3.14/site-packages (from -r requirements.txt (line 32)) (1.76.0)
Collecting grpcio-status==1.76.0 (from -r requirements.txt (line 34))
  Downloading grpcio_status-1.76.0-py3-none-any.whl.metadata (1.1 kB)
Requirement already satisfie
...


# Start the server
➜  emailservice git:(main) ✗ python email_server.py
{"timestamp": 1785786940.996674, "severity": "INFO", "name": "emailservice-server", "message": "starting the email service in dummy mode."}
{"timestamp": 1785786940.996759, "severity": "INFO", "name": "emailservice-server", "message": "Profiler enabled."}
{"timestamp": 1785786940.996796, "severity": "INFO", "name": "emailservice-server", "message": "Tracing disabled."}
{"timestamp": 1785786941.005613, "severity": "INFO", "name": "emailservice-server", "message": "listening on port: 7000"}
WARNING: All log messages before absl::InitializeLog() is called are written to STDERR
E0000 00:00:1785786941.007585 3676180 add_port.cc:83] Failed to add port to server: No address added out of total 1 resolved for '[::]:7000'
Traceback (most recent call last):
  File "/Users/aankansah/Code/akansof/10-downloads/shopease/src/emailservice/email_server.py", line 200, in <module>
    start(dummy_mode = True)
    ~~~~~^^^^^^^^^^^^^^^^^^^
  File "/Users/aankansah/Code/akansof/10-downloads/shopease/src/emailservice/email_server.py", line 131, in start
    server.add_insecure_port('[::]:'+port)
    ~~~~~~~~~~~~~~~~~~~~~~~~^^^^^^^^^^^^^^
  File "/Library/Frameworks/Python.framework/Versions/3.14/lib/python3.14/site-packages/grpc/_server.py", line 1461, in add_insecure_port
    return _common.validate_port_binding_result(
           ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~^
        address, _add_insecure_port(self._state, _common.encode(address))
        ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    )
    ^
  File "/Library/Frameworks/Python.framework/Versions/3.14/lib/python3.14/site-packages/grpc/_common.py", line 179, in validate_port_binding_result
    raise RuntimeError(_ERROR_MESSAGE_PORT_BINDING_FAILED % address)
RuntimeError: Failed to bind to address [::]:7000; set GRPC_VERBOSITY=debug environment variable to see detailed error message.
```

## Frontend Service

```Shell
# bulid the frontend
➜  frontend git:(main) ✗ go build -o frontend .


# ensure all required environment variables are on the frontend
PRODUCT_CATALOG_SERVICE_ADDR=localhost:3550
CURRENCY_SERVICE_ADDR=localhost:7000
CART_SERVICE_ADDR=localhost:7070
RECOMMENDATION_SERVICE_ADDR=localhost:8080
SHIPPING_SERVICE_ADDR=localhost:50051
CHECKOUT_SERVICE_ADDR=localhost:5050
AD_SERVICE_ADDR=localhost:9555

# then start it
./frontend
```


## Load Generator

```
# Install load generator app
python3 -m pip install -r requirements.txt

# Start load generator app
python -m locust -f locustfile.py --host=http://localhost:8088 --headless -u 10 -r 1

# =============
# Output
# =============

➜  loadgenerator git:(main) ✗ python -m locust -f locustfile.py --host=http://localhost:8088 --headless -u 10 -r 1
[2026-08-04 18:10:57,810] Augustines-MacBook-Pro/INFO/locust.main: Starting Locust 2.43.0
[2026-08-04 18:10:57,852] Augustines-MacBook-Pro/INFO/locust.main: No run time limit set, use CTRL+C to interrupt
Type     Name  # reqs      # fails |    Avg     Min     Max    Med |   req/s  failures/s
--------||-------|-------------|-------|-------|-------|-------|--------|-----------
--------||-------|-------------|-------|-------|-------|-------|--------|-----------
         Aggregated       0     0(0.00%) |      0       0       0      0 |    0.00        0.00

[2026-08-04 18:10:57,853] Augustines-MacBook-Pro/INFO/locust.runners: Ramping to 10 users at a rate of 1.00 per second
Type     Name  # reqs      # fails |    Avg     Min     Max    Med |   req/s  failures/s
--------||-------|-------------|-------|-------|-------|-------|--------|-----------
GET      /          2     0(0.00%) |    151      29     272     30 |    0.00        0.00
GET      /cart       1     0(0.00%) |     17      17      17     17 |    0.00        0.00
GET      /product/2ZYFJ3GM2N       1     0(0.00%) |     32      32      32     32 |    0.00        0.00
--------||-------|-------------|-------|-------|-------|-------|--------|-----------
         Aggregated       4     0(0.00%) |     88      17     272     30 |    0.00        0.00

Type     Name  # reqs      # fails |    Avg     Min     Max    Med |   req/s  failures/s
--------||-------|-------------|-------|-------|-------|-------|--------|-----------
GET      /          4     0(0.00%) |     84      15     272     18 |    1.00        0.00
GET      /cart       2     0(0.00%) |     13       9      17     10 |    0.50        0.00
POST     /cart       1     0(0.00%) |     19      19      19     19 |    0.00        0.00
GET      /product/2ZYFJ3GM2N       1     0(0.00%) |     32      32      32     32 |    0.50        0.00
GET      /product/9SIQT8TOJO       1     0(0.00%) |      7       7       7      7 |    0.00        0.00
GET      /product/OLJCESPC7Z       1     0(0.00%) |     10      10      10     10 |    0.00        0.00
--------||-------|-------------|-------|-------|-------|-------|--------|-----------
         Aggregated      10     0(0.00%) |     43       7     272     17 |    2.00        0.00

Type     Name  # reqs      # fails |    Avg     Min     Max    Med |   req/s  failures/s
--------||-------|-------------|-------|-------|-------|-------|--------|-----------
GET      /          6     0(0.00%) |     63      15     272     21 |    1.00        0.00
GET      /cart       4     0(0.00%) |      9       5      17      6 |    0.25        0.00
POST     /cart       1     0(0.00%) |     19      19      19     19 |    0.25        0.00
GET      /product/2ZYFJ3GM2N       1     0(0.00%) |     32      32      32     32 |    0.25        0.00
GET      /product/9SIQT8TOJO       1     0(0.00%) |      7       7       7      7 |    0.25        0.00
GET      /product/OLJCESPC7Z       1     0(0.00%) |     10      10      10     10 |    0.25        0.00
POST     /setCurrency       1     0(0.00%) |     10      10      10     10 |    0.00        0.00
--------||-------|-------------|-------|-------|-------|-------|--------|-----------
         Aggregated      15     0(0.00%) |     33       5     272     17 |    2.25        0.00

Type     Name  # reqs      # fails |    Avg     Min     Max    Med |   req/s  failures/s
--------||-------|-------------|-------|-------|-------|-------|--------|-----------
GET      /          8     0(0.00%) |     50      13     272     18 |    1.00        0.00
GET      /cart       5     0(0.00%) |      8       4      17      6 |    0.60        0.00
POST     /cart       2     0(0.00%) |     25      19      31     20 |    0.20        0.00
GET      /product/2ZYFJ3GM2N       1     0(0.00%) |     32      32      32     32 |    0.20        0.00
GET      /product/9SIQT8TOJO       1     0(0.00%) |      7       7       7      7 |    0.20        0.00
GET      /product/L9ECAV7KIM       1     0(0.00%) |      6       6       6      6 |    0.00        0.00
GET      /product/OLJCESPC7Z       1     0(0.00%) |     10      10      10     10 |    0.20        0.00
POST     /setCurrency       1     0(0.00%) |     10      10      10     10 |    0.00        0.00
--------||-------|-------------|-------|-------|-------|-------|--------|-----------
         Aggregated      20     0(0.00%) |     28       4     272     15 |    2.40        0.00

[2026-08-04 18:11:06,888] Augustines-MacBook-Pro/INFO/locust.runners: All users spawned: {"WebsiteUser": 10} (10 total users)
Type     Name  # reqs      # fails |    Avg     Min     Max    Med |   req/s  failures/s
--------||-------|-------------|-------|-------|-------|-------|--------|-----------
GET      /         11     0(0.00%) |     40       8     272     18 |    1.00        0.00
GET      /cart       7     0(0.00%) |      7       4      17      6 |    0.62        0.00
POST     /cart       4     0(0.00%) |     18       7      31     16 |    0.25        0.00
GET      /product/1YMWWN1N4O       1     0(0.00%) |     10      10      10     10 |    0.00        0.00
GET      /product/2ZYFJ3GM2N       1     0(0.00%) |     32      32      32     32 |    0.12        0.00
GET      /product/9SIQT8TOJO       1     0(0.00%) |      7       7       7      7 |    0.12        0.00
GET      /product/L9ECAV7KIM       2     0(0.00%) |      6       5       6      6 |    0.12        0.00
GET      /product/OLJCESPC7Z       3     0(0.00%) |      9       6      11     11 |    0.12        0.00
POST     /setCurrency       1     0(0.00%) |     10      10      10     10 |    0.12        0.00
--------||-------|-------------|-------|-------|-------|-------|--------|-----------
         Aggregated      31     0(0.00%) |     21       4     272     12 |    2.50        0.00

Type     Name  # reqs      # fails |    Avg     Min     Max    Med |   req/s  failures/s
--------||-------|-------------|-------|-------|-------|-------|--------|-----------
GET      /         11     0(0.00%) |     40       8     272     18 |    1.10        0.00
GET      /cart       7     0(0.00%) |      7       4      17      6 |    0.60        0.00
POST     /cart       5     0(0.00%) |     16       6      31     16 |    0.20        0.00
POST     /cart/checkout       1     0(0.00%) |     63      63      63     63 |    0.00        0.00
GET      /product/1YMWWN1N4O       2     0(0.00%) |      9       8      10      8 |    0.00        0.00
GET      /product/2ZYFJ3GM2N       1     0(0.00%) |     32      32      32     32 |    0.10        0.00
GET      /product/9SIQT8TOJO       1     0(0.00%) |      7       7       7      7 |    0.10        0.00
GET      /product/L9ECAV7KIM       2     0(0.00%) |      6       5       6      6 |    0.20        0.00
GET      /product/LS4PSXUNUM       1     0(0.00%) |     12      12      12     12 |    0.00        0.00
GET      /product/OLJCESPC7Z       4     0(0.00%) |      9       6      11     10 |    0.10        0.00
POST     /setCurrency       2     0(0.00%) |     11      10      11     10 |    0.10        0.00
--------||-------|-------------|-------|-------|-------|-------|--------|-----------
         Aggregated      37     0(0.00%) |     21       4     272     12 |    2.50        0.00

Type     Name  # reqs      # fails |    Avg     Min     Max    Med |   req/s  failures/s
--------||-------|-------------|-------|-------|-------|-------|--------|-----------
GET      /         11     0(0.00%) |     40       8     272     18 |    0.90        0.00
GET      /cart       7     0(0.00%) |      7       4      17      6 |    0.60        0.00
POST     /cart       6     0(0.00%) |     14       6      31      8 |    0.50        0.00
POST     /cart/checkout       2     0(0.00%) |     37      10      63     10 |    0.10        0.00
GET      /product/0PUK6V6EV0       1     0(0.00%) |      6       6       6      6 |    0.00        0.00
GET      /product/1YMWWN1N4O       2     0(0.00%) |      9       8      10      8 |    0.20        0.00
GET      /product/2ZYFJ3GM2N       1     0(0.00%) |     32      32      32     32 |    0.00        0.00
GET      /product/9SIQT8TOJO       1     0(0.00%) |      7       7       7      7 |    0.10        0.00
GET      /product/L9ECAV7KIM       2     0(0.00%) |      6       5       6      6 |    0.20        0.00
GET      /product/LS4PSXUNUM       1     0(0.00%) |     12      12      12     12 |    0.00        0.00
GET      /product/OLJCESPC7Z       4     0(0.00%) |      9       6      11     10 |    0.40        0.00
POST     /setCurrency       3     0(0.00%) |     12      10      14     12 |    0.10        0.00
--------||-------|-------------|-------|-------|-------|-------|--------|-----------
         Aggregated      41     0(0.00%) |     20       4     272     11 |    3.10        0.00

Type     Name  # reqs      # fails |    Avg     Min     Max    Med |   req/s  failures/s
--------||-------|-------------|-------|-------|-------|-------|--------|-----------
GET      /         11     0(0.00%) |     40       8     272     18 |    0.90        0.00
GET      /cart       7     0(0.00%) |      7       4      17      6 |    0.60        0.00
POST     /cart       6     0(0.00%) |     14       6      31      8 |    0.50        0.00
POST     /cart/checkout       2     0(0.00%) |     37      10      63     10 |    0.10        0.00
GET      /product/0PUK6V6EV0       1     0(0.00%) |      6       6       6      6 |    0.00        0.00
GET      /product/1YMWWN1N4O       2     0(0.00%) |      9       8      10      8 |    0.20        0.00
GET      /product/2ZYFJ3GM2N       1     0(0.00%) |     32      32      32     32 |    0.00        0.00
GET      /product/9SIQT8TOJO       1     0(0.00%) |      7       7       7      7 |    0.10        0.00
GET      /product/L9ECAV7KIM       2     0(0.00%) |      6       5       6      6 |    0.20        0.00
GET      /product/LS4PSXUNUM       1     0(0.00%) |     12      12      12     12 |    0.00        0.00
GET      /product/OLJCESPC7Z       4     0(0.00%) |      9       6      11     10 |    0.40        0.00
```
