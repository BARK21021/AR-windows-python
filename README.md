# 医学手术视频交互学习系统（Windows / Android 双端）

把一段完整的手术教学视频切成若干片段，**每个片段播完就地弹出一道题**——四选一的解剖/术式判断，或者「把标签拖到图上正确位置」的标注题。答对才能进入下一段，答错给出正确答案与讲解，学完给出正确率与学习进度。

一套内容数据、两份前端实现：

| 端 | 技术栈 | 入口 | 产物 |
|---|---|---|---|
| Windows 桌面端 | PyQt5 + OpenCV | `main.py` | `python main.py` 直接运行；可双击 `run_windows.bat` 一键建环境并启动 |
| Android 移动端 | Kivy + Buildozer | `main_android.py` | `./build_android.sh` → `bin/*.apk`（arm64-v8a / armeabi-v7a，min API 33） |

## 它解决什么问题

手术教学的痛点不在于没有视频，而在于**看完没记住**。一段 20 分钟的手术录像，学员常常只在「老师讲到哪」的位置暂停一下，无法验证自己真的看懂了关键解剖结构。

这个系统把「看视频」变成「边看边验」：

- **强制分段**：一段视频只讲一个步骤，播完立刻提问，不让学员一口气刷完；
- **两种题型对应两种能力**：选择题考判断（诊断、术式），标注题考空间认知（瓣叶分区、切除范围、标记线位置）；
- **错了必须看讲解**：答错不跳过，直接给出正确答案与解剖学解释，再继续；
- **进度与正确率本地留存**：不依赖任何后端服务，断网、离线、装在教室平板里都能用。

## 学习流程

```
选择视频（缩略图 + 标题 + 描述）
      ↓
视频片段按钮列表（每段标注时间点，有题的段落带问号标记）
      ↓ 选中一段
OpenCV 逐帧解码 → 显示在窗口里（播放 / 暂停 / 停止 / 拖动进度条 / 全屏）
      ↓ 该片段播放结束
片段带题？──否──→ 直接进入下一段
      │
      是
      ↓
弹出题目
   ├─ choice：四个选项，单选提交
   └─ annotation：把标签拖到图上对应位置，全部拖入图内才允许提交
      ↓
判分 → 正确：绿色反馈 + 继续；错误：红色反馈 + 正确答案 + 解剖学讲解
      ↓
全部片段学完 → 本视频正确率 + 学习进度 + 「返回列表」
```

## 模块划分

| 文件 | 行数 | 职责 |
|---|---:|---|
| `main.py` | 397 | 主窗口与三段式页面栈（视频选择页 / 播放页 / 页脚），题目调度、判分、正确率与进度统计 |
| `video_player.py` | 392 | `VideoThread(QThread)` 用 `cv2.VideoCapture` 逐帧读取并 `frame_ready` 发信号；`VideoPlayerWidget` 负责画面渲染、进度条、片段切换 |
| `question_dialog.py` | 403 | `QuestionDialog`：选择题单选组 + 反馈态；`AnnotationCanvas` / `DraggableLabel`：拖拽标注题，标签始终绘制在最上层，越界自动回弹 |
| `data_manager.py` | 183 | 内容数据源：视频列表、片段划分、题目、答案与讲解，全部是 Python 字面量 dict，路径相对 `源预制件/` |
| `main_android.py` | 627 | Kivy 版同一套流程：`ScreenManager` 切主/播放屏，`AnnotationArea` 实现拖拽标注，`QuestionPopup` 实现选择题与反馈 |

## 内容数据格式

`data_manager.py` 里每个视频是一个 dict，`video_segments` 里每个片段有两种题型：

**选择题**

```python
{
    'video_file': os.path.join(self.source_path, '1-2.mp4'),
    'has_question': True,
    'time_display': "6'40''",          # 仅用于按钮上标注时间点
    'type': 'choice',
    'question': '根据注水实验，你认为腱索应该如何调整：',
    'image': os.path.join(self.source_path, '标题', '1.2.png'),
    'options': ['A. 不调整', 'B. 调整短一些', 'C. 调整长一些', 'D. 还需要进一步注水实验判断'],
    'answer': 'B',
    'explanation': '注水实验显示瓣叶对合不良，需要将腱索调整短一些以改善对合',
}
```

**标注题**（`answer` 固定为 `annotation_complete`，判定标准是全部标签都被拖进图像区域）

```python
{
    'video_file': os.path.join(self.source_path, '1-1.mp4'),
    'has_question': True,
    'time_display': "1'9''",
    'type': 'annotation',
    'question': '请在图中标记出二尖瓣A1,A2,A3,P1,P2,P3位置，并指出断裂腱索',
    'image': os.path.join(self.source_path, '1.1.png'),
    'annotation_items': [
        {'name': 'A1', 'image': os.path.join(self.source_path, '标题', 'A1.png')},
        # … A2 / A3 / P1 / P2 / P3 / 断裂腱索
    ],
    'answer': 'annotation_complete',
    'explanation': '二尖瓣分为前叶(A)和后叶(P)，各分为三个区段：A1/A2/A3和P1/P2/P3',
    'answer_image': os.path.join(self.source_path, '1.1answer.png'),   # 可选：标准答案图
}
```

片段不带题时用 `'has_question': False` + `'description'`，播完直接进下一段。

## 媒体文件放在哪

源码里**不含任何视频与图片**（体积与素材版权原因）。运行时按 `data_manager.py` 里的路径去 `源预制件/` 取文件，目录长这样：

```
源预制件/
├── 1.1.png              # 视频 1 缩略图 / 标注题底图
├── 1-1.mp4              # 视频 1 的第 1 段
├── 1-2.mp4
├── 1.1answer.png        # 标注题标准答案图（可选）
├── 2.1.png
├── 2-1.mp4
├── …
└── 标题/
    ├── A1.png  A2.png  A3.png  P1.png  P2.png  P3.png
    ├── 断裂腱索.png（duanlie.png）
    └── 右.png  左.png  无.png   red.png
```

换成自己的课程内容有两种做法：按上面的结构放文件，或者直接改 `data_manager.py` 里的 `videos_data` 列表——它是一个普通 Python list，加一个 dict 就多一个视频。

## 怎么跑

**Windows**

```bat
python -m venv venv
venv\Scripts\activate
pip install -r requirements_windows.txt
python main.py
```

或者直接双击 `run_windows.bat`（自动建虚拟环境、装依赖、启动）。依赖只有四个：`PyQt5` / `opencv-python` / `numpy`。

**Android**

```bash
pip install buildozer
./build_android.sh        # 内部执行 buildozer android debug
# 产物：bin/medicaltutorial-1.0.0-arm64-v8a_armeabi-v7a-debug.apk
```

`buildozer.spec` 已配好 `package.domain = org.medical`、`android.api = 33`、`orientation = landscape`、包含扩展名 `py,png,jpg,kv,atlas,mp4`。

## 目录结构

```
.
├── main.py                    # Windows 入口（PyQt5）
├── main_android.py            # Android 入口（Kivy）
├── video_player.py            # 视频解码线程与播放控件
├── question_dialog.py         # 选择题对话框 + 拖拽标注画布
├── data_manager.py            # 内容数据（视频 / 片段 / 题目 / 答案 / 讲解）
├── run_windows.bat            # Windows 一键启动
├── build_android.sh           # Android 打包脚本
├── buildozer.spec             # Buildozer 配置
├── requirements_windows.txt
├── requirements_android.txt
└── 源预制件/                   # 媒体素材（不入库，需自行放入）
```

## 技术栈

Python 3 · PyQt5 · OpenCV · NumPy · Kivy / KivyMD · Buildozer · QThread

## 说明

- 本仓是课程项目的源码副本：**只含代码，不含手术视频、患者影像与教学图片**。要跑起来需要自备素材放到 `源预制件/`。
- 示例数据里的解剖结构（二尖瓣 A1–P3 分区、主动脉瓣三瓣叶、左心耳切除）取自公开的心外科教材与期刊图，用于演示题目结构，不代表任何具体病例。
- 桌面端与移动端是两套独立实现（PyQt5 / Kivy），共享同一份内容数据设计，不共享 UI 代码。
