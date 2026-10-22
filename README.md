# MelodyRelay

歌曲到乐谱的 Web 产品。当前实现按 [1.0.0 需求、架构及全部设计图](docs/1.0.0/README.md) 开发，使用本机 PostgreSQL、Redis、私有文件和 Mailpit。账户、上传、真实 Agent、谱面播放和导出可在本地运行；**一键取得可用口风琴成谱尚未完成**。真实歌曲仍有短音、尾段及声部覆盖问题，自动验收还缺独立下拍和非静音的非目标声部分类生产能力，不能仅归为发布测试未做。尚未提交或部署，详细边界见[版本验证摘要](docs/1.0.0/README.md)。

本次加强了 Agent 的自动交付：持久任务事实、有限重试、收尾预算和同次任务校验；恢复时复用已记账响应。开启乐器适配时，可在创建页提前授权“来源谱需要核对时，也生成带标记的适配草稿”。未识别和不确定内容仍保留标记。详细能力与验证边界维护在版本说明中。

## 本机启动

需要 Python 3.11、uv、Node.js 24、pnpm、PostgreSQL 17、Redis、FFmpeg、FluidSynth、Mailpit。当前机器已安装这些依赖，三个本地数据库也已创建，直接执行以下步骤即可；启动器优先使用 Codex 已安装的 Node 24。不要使用旧的 `melodyrelay` 数据库。

更新依赖锁后先运行 `uv sync --project melodyrelay-api --python 3.11 --frozen --extra dev`。人声单旋律的有限候选复核现使用本地PyTorch/CREPE full；权重随锁定包安装，无需额外API key或运行时下载。首次安装的磁盘占用和下载量会增加，本机优先MPS、其它环境使用CPU；它只为候选提供独立证据，不能代替人工音乐真值。

持续音补证还使用 `torchfcpe==0.0.4` 随包MIT模型（约43 MB，CPU、安全加载及固定SHA）。它只在CREPE没有强反证、频谱/声部归属和同源人声上下文均满足时恢复原候选延音，不生成新起音。更新后同时重启API和worker；旧作品保持不变，新算法须从“调整识别方式，再试一次”生成新作品。

人声未知片段的初筛还需要两份固定PANNs离线权重（合计约351 MB）。首次或更新后运行 `uv run --frozen --project melodyrelay-api --env-file .env python scripts/cache_audio_models.py`，随后加 `--check` 可只验证、不下载。本机已准备好；权重放在私有存储相邻的 `models/panns/`，运行任务时不自动下载。代码为MIT、权重为CC-BY-4.0，来源及SHA随证据保存。初筛显示“可能无人声／可能有人声／暂不能判断”，不会把模型概率变成已确认休止或音乐准确率。

局部漏音对照新增固定 MelBand 人声模型（约913 MB，CPU，MIT权重及实现）。更新依赖后执行 `uv run --frozen --project melodyrelay-api --env-file .env python scripts/cache_audio_models.py --melband`，检查时追加 `--check`。本机已准备；模型目录为私有存储相邻的 `models/melband/`。Agent仅对已有主旋律的8–30秒疑难窗调用它，随后以默认Basic Pitch取候选并用原固定音源独立验证；不会覆盖整曲分轨、修改旧谱或在任务中下载权重。缺少权重会明确失败。更新后须同时重启API和worker。

默认音符转写现使用本地固定 Basic Pitch ONNX（CPU，约 0.8 MB 权重），准备命令为 `uv run --frozen --project melodyrelay-api --env-file .env python scripts/cache_audio_models.py --basic-pitch`，追加 `--check` 只验 SHA。本机已准备；不会在任务中下载。100ms 最短时长按模型实际时间戳判断，不再因帧数四舍五入漏掉 100–116ms 候选。同一任务、同一音源和完整参数的候选可复用，不产生该阶段的供应商费用。识别音符仍须通过独立质量检查。

缺拍复检使用 `beat-this==1.1.0` 与固定final0权重（约81 MB，MIT）。执行 `uv run --frozen --project melodyrelay-api --env-file .env python scripts/cache_audio_models.py --beat-this` 准备；追加 `--check` 只验SHA。本机已准备。最多检查四个40秒鼓轨上下文，结合原混音起音，保留未证实缺拍，不自动补成规则节奏。逐音验收复用120秒分段的CREPE/FCPE帧；音乐内容改变后重算验收，失败测量不会当成可复用成功。此能力不表示当前真实歌曲已经达到一键成谱质量。

1. 在根目录编辑 [`.env`](.env)；已放好本机默认地址和空白密钥，文件权限为0600。新机器若没有此文件，先复制 [`.env.example`](.env.example) 为 `.env`。不要覆盖已有配置，不要将真实密钥提交到 Git。
2. `OPENROUTER_API_KEY` 和 `REPLICATE_API_TOKEN` 用于真实音乐任务；当前终端已有 `FUBB_OPENROUTER_API_KEY` / `FUBB_REPLICATE` 时可留空，服务端会沿用。`RESEND_API_KEY` 本地留空，验证码在 Mailpit 查看。以后填写 Resend 时同时使用已验证域名的 `RESEND_FROM`。
3. 从项目根目录运行下面的命令。`uv --env-file` 负责读取根 `.env`，现有启动器负责向各服务传递需要的变量；不要直接用 `python3 scripts/dev.py` 期待它自动读取根 `.env`。

```sh
cd /Users/neurobin/my/workspace/MelodyRelay
brew services start postgresql@17
brew services start redis
uv run --frozen --project melodyrelay-api --env-file .env python scripts/dev.py --check
uv run --frozen --project melodyrelay-api --env-file .env python scripts/dev.py
```

启动后打开：

- 产品：[http://127.0.0.1:3000/zh-CN](http://127.0.0.1:3000/zh-CN)
- 本地验证码收件箱：[Mailpit](http://127.0.0.1:8025)
- API 契约：[OpenAPI](http://127.0.0.1:8000/docs)

启动器仅允许本机数据库和Redis地址，执行 Alembic 后启动 API、DBOS worker、PDF renderer、Web 和 Mailpit。保留这个终端查看日志；Ctrl-C 只终止它创建的进程。若提示端口已有进程或worker已由其他启动器管理，会保留该服务，不会替换配置；修改 `.env` 后，先在旧启动窗口Ctrl-C停止相应服务，再重新启动，尤其要同时重启API和worker。单独启动可在命令末尾追加 `--service api|worker|renderer|web|mailpit`，前端生产构建预览追加 `--web-mode start`（需先完成 `pnpm build`）。

本机worker的互斥锁由实际worker进程持有；即使启动器意外退出，仍存活的worker也会阻止重复消费者。正常Ctrl-C或SIGTERM会清理本次启动器创建的进程组。进程锁仅用于本地启动，不改变正式环境的DBOS并发策略。

配置优先级为同名终端环境变量 > 根 `.env` > 程序默认值；若文件里的修改没有生效，先检查是否仍有同名export或旧服务。现有环境中的密钥不会被复制到文件；前端不继承供应商或其他项目数据库凭据。排查进程时不要打印完整环境、进程命令行或进程标题。

上传策略唯一来源为 [`shared/upload-policy.json`](shared/upload-policy.json)：单文件200 MiB、multipart代理容量201 MiB、API处理期限200秒、浏览器上传与完成检查共用240秒总期限；Next上游socket静默超时为240秒。修改策略后须重新构建Web并重启API；部署或复制源码时必须同时保留根目录的`shared/`。旧`UPLOAD_MAX_BYTES`仅接受与共享策略相同的值，不能单独覆盖后端限制。超时、超限和不可解码分别明确提示，不自动启动音乐任务。

### 新机器首次准备

当前机器无需重复执行。其他机器先安装上述系统依赖，并在Node.js 24的终端安装锁定依赖、创建三个全新本地数据库；不要对已有库重复创建或清空。

```sh
cd /Users/neurobin/my/workspace/MelodyRelay
createdb melodyrelay_dev_100
createdb melodyrelay_dbos_dev_100
createdb melodyrelay_test_100
uv sync --project melodyrelay-api --python 3.11 --frozen --extra dev
pnpm --dir melodyrelay-web install --frozen-lockfile
pnpm --dir melodyrelay-renderer install --frozen-lockfile
pnpm --dir melodyrelay-renderer exec playwright install chromium
```

随后复制 `.env.example` 为 `.env`、填写配置，运行上方的 `cache_audio_models.py` 准备离线权重，再按上方启动。当前使用OpenRouter Sonnet 5.5和注册表固定版本的Replicate模型；真实音乐任务会产生供应商费用。Resend接口已实现，正式投递等待key和验证域名。

`scripts/dev.py` 默认使用 `melodyrelay_dev_100`、`melodyrelay_dbos_dev_100`、Redis DB 3；可通过 `DATABASE_URL`、`DBOS_DATABASE_URL`、`REDIS_URL` 显式覆盖本机地址。私有数据和自动生成的开发密钥位于 `~/.local/share/melodyrelay/`，不写入仓库。`SOUNDFONT_PATH` 默认指向此目录的 `soundfonts/FluidR3_GM.sf2`；当前已安装固定哈希的 FluidR3_GM，并保留 MIT 许可及来源清单。迁移到其他机器时应复制该受许可音色和清单，或显式配置同一音色路径。

## 代码入口

| 目录 | 内容 |
| --- | --- |
| `melodyrelay-web/` | Next.js 产品页面、播放器、编辑与校对、后台、共享简谱/VexFlow 打印组件 |
| `melodyrelay-api/` | FastAPI、账户与权限、Score/Patch/TimeMap、真实 Agent、音频工具、账本、DBOS 工作流 |
| `melodyrelay-renderer/` | 受控 Score→Chromium A4 PDF；固定字体、鉴权、资源限制 |
| `scripts/` | 本机启动、契约生成、明确付费的原创探针、人工标注集评测 |
| `shared/` | Web、Next代理及API共同读取的上传限制 |
| `docs/1.0.0/` | 本版本唯一需求、架构、设计与实施验证摘要 |

前端保留自己的 Git 边界，根目录没有初始化 Git。原有参考歌曲未移动；任何原创示例都明确标识，不作为识别准确率证据。

练习音频生命周期入口为 `melodyrelay-api/app/audio/practice.py`，用户/Agent不可变版本写入入口为 `melodyrelay-api/app/core/versions.py`，播放器资源生命周期入口为 `melodyrelay-web/src/features/music/playback-session.ts`。HTTP、工具与页面保留自己的授权/展示语义；规则及回归范围见[技术方案](docs/1.0.0/architecture.md)和[版本验证摘要](docs/1.0.0/README.md)。

处理页按音乐阶段显示概览，完整工具记录在“处理依据与技术记录”。结果先试听，再按片段查看局部问题；覆盖不足会明确显示系统识别限制。可从结果选择“调整识别方式，再试一次”，复用原音、调整声部/片段并重新授权，新建作品保留旧稿。详细质量边界和本地验证见 [1.0.0说明](docs/1.0.0/README.md)。

## 验证

```sh
cd melodyrelay-api
uv run --frozen pytest -q
uv run --frozen alembic check
cd ../melodyrelay-web
pnpm test
pnpm test:upload-proxy
pnpm typecheck
pnpm lint
pnpm build
cd ../melodyrelay-renderer
pnpm test
pnpm exec tsc --noEmit
```

API 测试只允许隔离的 `melodyrelay_test_100`，每例回滚且私有文件落入临时目录。`test:upload-proxy`启动独立Next和受控上游，覆盖原10 MiB边界、200 MiB加multipart、35秒静默响应及断连恢复，约40秒，不占用现有3000端口。

服务启动后，从项目根目录执行不产生供应商费用的浏览器上传验收：

```sh
uv run --frozen --project melodyrelay-api --env-file .env python scripts/verify_uploads.py \
  --output /tmp/melodyrelay-upload-acceptance/acceptance.json
```

工具用独立Chromium和短期普通账号会话，七首现有完整歌曲、200 MiB−1字节与恰好200 MiB的合法FLAC均从真实创建页上传、完成检查、试听与Range读取；随后逐文件核对数据库、私有文件及经过Next完整回读的SHA。超过上限1字节同时验证页面拦截和服务端413。纯文本413/504与240秒期限采用明确的受控响应/虚拟时钟，验证错误与恢复状态，不能当作真实长时间处理或音乐识别验收。只清理本次工具创建且无任务引用的账号、会话与上传，不创建作品或供应商调用；不保存报告时可省略`--output`。

真实付费链路与标注评测单独执行：

```sh
uv run --project melodyrelay-api --frozen python scripts/probe-agent.py --paid --source-part original
uv run --project melodyrelay-api --frozen python scripts/evaluate_music.py /absolute/path/to/annotations.json
uv run --project melodyrelay-api --frozen python scripts/verify_real_songs.py --paid \
  --samples /absolute/path/to/assets/music \
  --output /absolute/path/outside/repository/acceptance.json
```

探针使用脚本生成的原创音频，实际调用供应商并记录费用；`--segment-start/--segment-end` 可验证片段，结果默认只输出终端。评测输入契约为 `app/domain/evaluation.py` 的 `EvaluationDataset`：人工金标准与实际 Score 配对，按 recording_id 隔离调参/留出集；未满足样本覆盖或质量门槛时退出码为 2。两者只有显式 `--output` 才保存结果。

真实歌曲工具使用现有七首录音覆盖四目标×适配开关，核对固定来源/适配版本的 MIDI、MusicXML、PDF、文件哈希和原曲未改变。它只在本机隔离开发库创建普通测试账号及短会话，音乐、适配和导出均经过普通用户真实 API，开发费用仅从本机内部账本只读获取，不暴露到产品界面；仍遵守单账号预算和并发限制。`--resume` 继续已有报告；`--case` 限定组合；`--sources-only` 先审来源；修复后 `--resume --case <组合> --rebuild-source` 保留历史并创建新来源，`--retry-failed` 只重试已失败的阶段，供应商在途或提交未知时禁止重建。`--batch-size 1` 可串行执行，以适应账户剩余预留预算，不改变服务端预算。交付通过必须包含当前来源及所需适配的全部三种格式，并核对任务、作品和来源引用；PDF排版仍须独立渲染复核。报告及下载文件仅在显式路径保存，歌曲、会话凭据和完整日志不放入仓库。这是链路及产物一致性验收，不能代替人工标注准确率或听辨，也不将短会话夹具算作注册登录验收。

用户提供的纸谱可用于**离线参考一致性**对照，不会成为用户导入歌曲的必填项，也不进入运行时识别/质量放行。当前两页《出现又离开》的独立目读样本位于 `melodyrelay-api/tests/fixtures/appear_leave_reference.json`；保留原图SHA、音级/上下点、时值、同音延音、倚音未定时及覆盖限制。对照入口：

```bash
uv run --project melodyrelay-api --frozen python scripts/compare_score_reference.py \
  --score /absolute/path/fixed-score.json \
  --reference melodyrelay-api/tests/fixtures/appear_leave_reference.json \
  --anchor-midi 57 --windows /absolute/path/windows.json
```

`windows.json` 是数组，每项明确 `phrase_id`、完整句的 `start_seconds/end_seconds`、单个 `voice_ids` 和 `complete_phrase: true`；可用乐句ID见fixture。`--anchor-midi` 只允许一次全局中音1锚点，不按句自由移调。脚本合并真实延音后做有序一对一对照，报告相对参考的缺音/多音/八度或音高差异、对齐歧义和拍数比例；没有录音秒级标注时不输出音符准确率。默认仅终端输出，显式 `--output /absolute/path/report.json` 才保存。图片与录音可能存在编配差异，报告不会修改Score、调用供应商或签发`accepted`。

Resend 真实投递、人工标注留出集、P-37D 实琴、20分钟完整录音与正式环境负载/存储验收尚未完成。软件链路通过不等于音乐准确率或发布验收通过，草稿保持 `review_required`。
