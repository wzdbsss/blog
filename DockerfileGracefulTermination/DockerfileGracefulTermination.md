# Dockerfile best practice for graceful termination

## exec form v.s. shell form
Refer to [Official Docker ENTRYPOINT](https://docs.docker.com/engine/reference/builder/#entrypoint) documentation, Docker ENTRYPOINT has two forms:

ENTRYPOINT ["executable", "param1", "param2"] (exec form, preferred)
ENTRYPOINT command param1 param2 (shell form)

| exec form                                                                                                                                                        | shell form                                                                                                                                                                                               |
| ---------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Command line arguments to docker run <image> will be appended after all elements in an exec form ENTRYPOINT, and will override all elements specified using CMD. | The shell form prevents any CMD or run command line arguments from being used, but has the disadvantage that your ENTRYPOINT will be started as a subcommand of /bin/sh -c, which does not pass signals. |
| executable will be the container’s PID 1 - and will receive Unix signals - so your executable will receive a SIGTERM from docker stop <container>.               | executable will not be the container’s PID 1 - and will not receive Unix signals - so your executable will not receive a SIGTERM from docker stop <container>.                                           |
| does not invoke a command shell, which means that normal shell processing does not happen, e.g. no environment variable expansion like ${SW_AGENT_ENABLE} etc.   | environment variable expansion by /bin/sh process                                                                                                                                                        |

If the executable need graceful termination the preferred form is exec form, however exec form cannot pass the environment variables as parameter to executable.

Shell form can  pass the environment variables as parameter to executable via sh context, however it won't be the container's PID 1 process which won't receive a SIGTERM from docker stop for executable graceful termination.

## Solution to pass environment variable for graceful termination
Refer to official doc [Shell form ENTRYPOINT example](https://docs.docker.com/engine/reference/builder/#shell-form-entrypoint-example), To ensure that docker stop will signal any long running ENTRYPOINT executable correctly, you need to remember to start it with exec as shell form ENTRYPOINT:
```
FROM ubuntu
ENTRYPOINT exec top -b
```

To enable SW_AGENT_ENABLE environment variable for your Java Dockerfile ENTRYPOINT, please follow below example:

### Before: exex form without environment variable as java executable parameter & be the container's PID 1 for graceful termination.
```
FROM openjdk:8-jdk-slim
 
ADD ./build/libs/app.jar app.jar
 
ENTRYPOINT ["java"]
 
CMD [ "-Djava.security.egd=file:/dev/./urandom", "-Dspring.profiles.active=secured,aliyun", "-jar", "/app.jar" ]
```

### After: shell form with environment variable  ${SW_AGENT_ENABLE} as java executable parameter & be the container's PID 1 for graceful termination.

```
FROM openjdk:8-jdk-slim
 
ADD ./build/libs/app.jar app.jar
 
ENTRYPOINT exec java ${SW_AGENT_ENABLE} -Djava.security.egd=file:/dev/./urandom -Dspring.profiles.active=secured,aliyun -jar /app.jar
```