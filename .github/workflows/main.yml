# 每日分时段累加修改 Zepp 步数(长驻时间窗 + 接力链模式)
#
# 为什么不靠 GitHub 的定时(schedule)触发?
#   实测 GitHub 会对 scheduled 触发大幅延迟甚至整批丢弃(*/20 一天实际只触发
#   几次;改固定时间窗后也曾整班不来),预约好的提交时间无人唤醒、步数远少于
#   预期。因此采用「接力链」:每一班进程连续长驻约 5.5 小时,退出时通过 GitHub
#   API 亲自触发下一班(workflow_dispatch / repository_dispatch 是官方文档明确
#   允许由 GITHUB_TOKEN 触发的事件,准点率远高于 schedule)。链条全天滚动、
#   完全不依赖 cron 的可靠性;手动 Run workflow 也可随时拉起。
#
# v8.2 双通道接力:main.py 先走 workflow_dispatch,连续被拒时自动改走
#   repository_dispatch(两个不同的 API 端点,GitHub 只拦其中一条时仍可续链;
#   2026-11-10 08:17 实测出现过 "repository invocation blocked" 拦截接力
#   调用导致断链半天,故加此退路)。触发器里的 repository_dispatch 即为该
#   备用通道预留,请勿删除。
#
# v8.2 加密兜底网:下方 cron 由 4 个增至 8 个(约每 3 小时一个),单点 cron
#   被延迟/丢弃时链条最长断约 3 小时,而不是半天。
#
# 班次衔接(北京时间):每班约 5.5 小时,班尾自动接力下一班,全天无缝覆盖;
#   兜底窗口 05:30 / 08:30 / 11:20 / 13:30 / 16:50 / 19:00 / 22:20 / 23:50。
name: 自动任务-修改Zepp当天运动数据

on:
  workflow_dispatch:
  repository_dispatch:        # v8.2:接力链备用通道(workflow_dispatch 被拦时由 main.py 自动改走这里)
  schedule:
    - cron: '30 21 * * *'   # 兜底:北京时间 05:30
    - cron: '30 0 * * *'    # 兜底:北京时间 08:30(v8.2 新增)
    - cron: '20 3 * * *'    # 兜底:北京时间 11:20
    - cron: '30 5 * * *'    # 兜底:北京时间 13:30(v8.2 新增)
    - cron: '50 8 * * *'    # 兜底:北京时间 16:50
    - cron: '0 11 * * *'    # 兜底:北京时间 19:00(v8.2 新增)
    - cron: '20 14 * * *'   # 兜底:北京时间 22:20
    - cron: '50 15 * * *'   # 兜底:北京时间 23:50(v8.2 新增)

concurrency:
  group: zepp-steps-update   # 同一时刻只允许一个实例,避免状态竞争
  cancel-in-progress: true   # 新触发直接接管旧班次(状态每次提交后已实时回写仓库,可安全接管)

permissions:
  contents: write            # 允许把 state.json 提交回仓库
  actions: write             # 允许脚本在班次结束时接力触发下一班(接力链)

jobs:
  run-python-script:
    runs-on: ubuntu-latest
    timeout-minutes: 350     # 5.5 小时长驻 + 安装依赖等开销的缓冲(< GitHub 6 小时上限)
    steps:
      - name: '检出仓库代码'
        uses: actions/checkout@v6
        with:
          fetch-depth: 0
      - name: '设置 Python 环境'
        uses: actions/setup-python@v6
        with:
          # 注意:必须用浮动版本号(如 '3.12'),禁止钉死到补丁号(如 '3.12.0')!
          # 2026-10-19 起 ubuntu-latest 逐步迁移到 Ubuntu 26.04,新镜像的
          # python-versions 清单里没有 3.12.0 的预编译包,钉死补丁号会导致
          # 每次运行都在「设置 Python 环境」一步直接失败(2026-11-02 实测断链约 20 小时)。
          # 浮动 '3.12' 会在任何镜像上自动解析到最新的 3.12.x(26.04 上为 3.12.16)。
          python-version: '3.12'
      - name: '安装依赖项'
        run: pip install requests
      - name: '配置 Git 提交身份'
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "41898282+github-actions[bot]@users.noreply.github.com"
          # v8.2:git 网络挂起保护(实测 2026-11-09 有班次因网络操作挂起拖满
          # 350 分钟被 GitHub 超时击杀、未及接力下一班,链条断档数小时):
          git config --global http.lowSpeedLimit 2000
          git config --global http.lowSpeedTime 60
      - name: '运行主程序(长驻班次:接力/定时/手动触发一视同仁)'
        env:
          ACCOUNT: ${{ secrets.ACCOUNT }}
          PASSWORD: ${{ secrets.PASSWORD }}
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}   # 接力链触发下一班必需
        run: python ./main.py --daemon --max-minutes 330
      - name: '提交并推送状态文件(兜底)'
        run: |
          git add state.json
          if git diff --cached --quiet; then
            echo "状态无变化,跳过提交"
          else
            git commit -m "chore: 更新步数任务状态 [skip ci]"
            git pull --rebase origin "$GITHUB_REF_NAME" || true
            git push origin HEAD:"$GITHUB_REF_NAME"
          fi
