### 安装依赖

yarn install
yarn run dev

用 Seedance 视频相关模型，建议额外执行：`npx playwright-core install chromium`

测试
http://localhost:8000/ping

python test-async-video.py 你的sessionid
成功后会打印 `task_id`，并轮询直到返回视频 URL
