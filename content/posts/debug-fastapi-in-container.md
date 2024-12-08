+++
title = "调试容器中的 FastAPI 应用"
author = ["4shen0ne"]
publishDate = 2024-12-08T00:00:00+08:00
tags = ["vscode", "fastapi"]
draft = false
+++

1.  镜像中需添加 [debugpy](https://github.com/microsoft/debugpy) 依赖库，并在启动容器时运行 debugpy 模块
    ```dockerfile
    FROM tiangolo/uvicorn-gunicorn-fastapi:python3.7
    # Note: In default, the working directory is /app in this base image.

    COPY requirements.txt .
    RUN pip install --no-cache-dir -r requirements.txt

    EXPOSE 5678
    EXPOSE 8000

    COPY ./app /app
    ENTRYPOINT ["python3", "-m", "debugpy", "--listen", "0.0.0.0:5678", "-m", "uvicorn", "main:app", "--reload", "--host", "0.0.0.0", "--port", "8000"]
    ```
    <div class="src-block-caption">
      <span class="src-block-number">Code Snippet 1:</span>
      Dockerfile
    </div>

2.  运行时连接 debugpy 服务端的监听端口，并设置本地和容器的项目路径映射
    ```json
    {
      "version": "0.2.0",
      "configurations": [
        {
          "name": "Python: FastAPI Container Debug",
          "type": "debugpy",
          "request": "attach",
          "connect": {
            "port": 5678,
            "host": "localhost",
          },
          "pathMappings": [
            {
              "localRoot": "${workspaceFolder}/app",
              "remoteRoot": "/app"
            }
          ]
        }
      ]
    }
    ```
    <div class="src-block-caption">
      <span class="src-block-number">Code Snippet 2:</span>
      .vscode/launch.json
    </div>

<!--more-->
