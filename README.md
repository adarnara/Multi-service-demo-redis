# CS 122B Project 5 Murphy movies example

- This example application allows you to log in, see a movie by Eddie Murphy and leave comments.

## This README.md file is used for Multi-Service

## Brief Explanation

- The Multi-Service branch uses Redis instead of session to store shared user information and star pod state.
- We modified the `pom.xml` to compile different part of the projects into different war files.

### Redis

We use Redis to replace pod-local session storage. A utility class `RedisUtil` is added to help you use Redis.
The changes in `LoginServlet` and `LoginFilter` show you how to replace session with Redis-backed state.
- `common/RedisUtil.java`: It contains functions to initialize Redis, `get`, `set`, and `incr`.
- `login/LoginServlet.java`: It shows how to create login state in Redis. The username and the loginTime are serialized and stored under `session:<sessionId>`. Then the `sessionId` is set to cookies, so later requests always contain the session id.
- `common/LoginFilter.java`: It shows how to get `sessionId` from cookies and how to validate login by reading session data from Redis.
- `star/SingleStarServlet.java`: It shows different way of storing states. `loginTime` is a state shared by both services, we store it in Redis session data. `accessCount` is a state for star service, and we store it in Redis with key `accessCount:<username>`, so all star pods share the same counter.

Redis address is configured in `WebContent/META-INF/context.xml`:
```xml
<Environment name="redis/Address" type="java.lang.String" value="redis-service:6379"/>
```

### Maven Profiles
The original Maven configuration will compile everything in the codebase into a war file. Using Maven Profiles, we can compile different part of project into different war files.
In this branch, we split `/api/login` endpoint to a login profile, and the other endpoints to a star profile.
- First, split the source files into different packages. Note that we have a package called `common`, which is shared among different profiles.
- Next, modify the `pom.xml` to set up different profiles.
  - Line 46: we change the sourceDirectory from `src` to a parameter `${endpointDir}`. You can set different value to this parameter for different profiles.
  - Line 61: we set a parameter `${excludes}`, which can be used to exclude some static files inside `WebContent`.
  - Line 64-81 show how to add common package to all profiles
  - Line 85-107 show how to use Profiles. For each profile we define the value of `endpointDir` and `excludes`.

The `Dockerfile` is also updated. At line 7 we defined an argument `MVN_PROFILE`, you can set its value when building an image.

## Build different profiles
- Compile different part of the project into war file with
  ```
  mvn package -P ${profileName}
  ```
- Login endpoint:
  ```
  mvn package -P login
  ```
- Star endpoints:
  ```
  mvn package -P star
  ```

If you see errors in Intellij, open the Maven panel at the right. Expand "Profiles" and select only "default". Then reload the Maven Project.

## Build different Docker images

- Build the image for login endpoint with
  ```
  sudo docker build . --build-arg MVN_PROFILE=login --platform linux/amd64 -t <DockerHub-user-name>/cs122b-p5-murphy-login:v1
  ```
  - We specify the Maven profile name with `--build-arg MVN_PROFILE=${profileName}`
- Push the image to DockerHub with
  ```
  sudo docker push <DockerHub-user-name>/cs122b-p5-murphy-login:v1
  ```
- Repeat the steps for star endpoint:
  ```
  sudo docker build . --build-arg MVN_PROFILE=star --platform linux/amd64 -t <DockerHub-user-name>/cs122b-p5-murphy-star:v1
  ```
  ```
  sudo docker push <DockerHub-user-name>/cs122b-p5-murphy-star:v1
  ```
