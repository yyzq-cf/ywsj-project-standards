# 多语言 (i18n) 支持参考实现

> 本文件是 ywsj-project-standards 第十一章的完整实现模板。
> 所有 Flask 项目默认支持中文(zh)和英文(en)，其他语言可按需扩展。

---

## 一、文件结构

```
project-name/
├── app.py
├── i18n/
│   ├── __init__.py          # i18n 核心逻辑
│   ├── translations.py      # 翻译字典
│   └── locales/             # (可选) 拆分到单独 JSON 文件
│       ├── zh.json
│       ├── en.json
│       └── ja.json          # 新增语言只需加文件
├── templates/
│   ├── base.html            # 包含语言切换器
│   ├── index.html
│   └── ...                  # 每个模板用 t() filter
└── static/
    └── js/
        └── i18n.js          # 前端 JS 翻译
```

**两种翻译存储方式：**

| 方式 | 适用场景 | 优点 | 缺点 |
|------|----------|------|------|
| `translations.py` 单文件字典 | 翻译条目 < 200 | 零文件IO、加载快 | 大量条目时不好维护 |
| `locales/*.json` 拆分文件 | 翻译条目 > 200 或多语言 | 每语言独立维护、可热加载 | 多一次文件读取 |

推荐：项目初期用 `translations.py`，语言超过3种或条目超过200时迁移到 `locales/`。

---

## 二、后端 Flask 实现

### 2.1 i18n/__init__.py — 核心逻辑

```python
"""i18n 多语言支持模块

用法:
    from i18n import get_locale, t, init_i18n
    init_i18n(app)  # 在 create_app() 中调用
"""
import json
import os
from pathlib import Path
from flask import session, request, g, current_app

# 支持的语言列表（默认 zh + en，可扩展）
SUPPORTED_LANGUAGES = {
    'zh': '中文',
    'en': 'English',
    # 新增语言只需取消注释或添加：
    # 'ja': '日本語',
    # 'ko': '한국어',
    # 'fr': 'Français',
    # 'de': 'Deutsch',
    # 'es': 'Español',
    # 'ru': 'Русский',
    # 'pt': 'Português',
}

DEFAULT_LANGUAGE = 'zh'
FALLBACK_LANGUAGE = 'en'  # 翻译缺失时 fallback 到英文


def _load_translations(translations_dir=None):
    """从 locales/ 目录加载所有语言的翻译 JSON"""
    translations = {}
    if translations_dir and Path(translations_dir).exists():
        for lang_file in Path(translations_dir).glob('*.json'):
            lang_code = lang_file.stem
            with open(lang_file, 'r', encoding='utf-8') as f:
                translations[lang_code] = json.load(f)
    return translations


def get_locale():
    """获取当前语言，优先级：session > cookie > Accept-Language > 默认"""
    # 1. session 中显式设置的语言（用户点语言切换器）
    lang = session.get('language')
    if lang and lang in SUPPORTED_LANGUAGES:
        return lang

    # 2. cookie 持久化（session 过期后仍记住）
    lang = request.cookies.get('language')
    if lang and lang in SUPPORTED_LANGUAGES:
        return lang

    # 3. Accept-Language header（首次访问自动检测浏览器语言）
    best = request.accept_languages.best_match(
        list(SUPPORTED_LANGUAGES.keys()),
        default=DEFAULT_LANGUAGE
    )
    return best or DEFAULT_LANGUAGE


def t(key, **kwargs):
    """翻译函数 — 后端 / Jinja2 模板通用

    用法:
        t('app.title')                        # → "字幕转文字工具"
        t('settings.api_key_hint', name='BCut')  # → "在BCut官网获取API Key"
    """
    translations = getattr(g, '_translations', None)
    if translations is None:
        translations = getattr(current_app, '_i18n_translations', {})
        g._translations = translations

    lang = get_locale()

    # 尝试当前语言 → fallback 英文 → 返回 key 本身
    text = _get_value(translations, lang, key)
    if text is None and lang != FALLBACK_LANGUAGE:
        text = _get_value(translations, FALLBACK_LANGUAGE, key)
    if text is None:
        return key  # 翻译缺失，返回 key 作为兜底

    # 模板变量替换 {name} {count} 等
    if kwargs:
        try:
            text = text.format(**kwargs)
        except (KeyError, IndexError):
            pass  # 格式化失败不影响返回原文

    return text


def _get_value(translations, lang, key):
    """从嵌套字典中按 a.b.c 路径取值"""
    data = translations.get(lang, {})
    for part in key.split('.'):
        if not isinstance(data, dict):
            return None
        data = data.get(part)
        if data is None:
            return None
    return data


def init_i18n(app):
    """初始化 i18n，在 create_app() 中调用

    Args:
        app: Flask app 实例
    """
    # 方式1: 从 translations.py 加载（推荐初期）
    try:
        from i18n.translations import TRANSLATIONS
        app._i18n_translations = TRANSLATIONS
    except ImportError:
        app._i18n_translations = {}

    # 方式2: 从 locales/ 目录加载（多语言扩展时）
    locales_dir = os.path.join(app.root_path, 'i18n', 'locales')
    if Path(locales_dir).exists():
        file_translations = _load_translations(locales_dir)
        # 合并：文件翻译覆盖内联翻译（方便覆盖个别 key）
        for lang, data in file_translations.items():
            if lang in app._i18n_translations:
                app._i18n_translations[lang].update(data)
            else:
                app._i18n_translations[lang] = data

    # 注册 Jinja2 全局函数和 filter
    app.jinja_env.globals['t'] = t
    app.jinja_env.globals['supported_languages'] = SUPPORTED_LANGUAGES
    app.jinja_env.globals['current_language'] = get_locale

    app.logger.info(f'i18n initialized: {list(app._i18n_translations.keys())}')


def set_language(lang_code):
    """设置当前语言（语言切换 API 调用）"""
    if lang_code not in SUPPORTED_LANGUAGES:
        return False
    session['language'] = lang_code
    return True
```

### 2.2 i18n/translations.py — 翻译字典

```python
"""翻译字典 — 默认中英文，其他语言按需扩展

结构: { lang_code: { namespace: { key: text } } }
Key 命名规范: namespace.key  (如 app.title, settings.api_key, common.submit)
"""

TRANSLATIONS = {
    # ==================== 中文（默认） ====================
    'zh': {
        'app': {
            'title': '字幕转文字工具',
            'subtitle': '支持多种ASR引擎的音视频转字幕服务',
        },
        'nav': {
            'home': '首页',
            'settings': '⚙️ 设置',
            'login': '登录',
            'logout': '退出',
        },
        'common': {
            'submit': '提交',
            'cancel': '取消',
            'save': '保存',
            'delete': '删除',
            'edit': '编辑',
            'loading': '加载中...',
            'success': '操作成功',
            'error': '操作失败',
            'confirm_delete': '确认删除？',
            'no_data': '暂无数据',
            'upload': '上传文件',
        },
        'settings': {
            'title': '设置',
            'tab_api': '🔑 API 配置',
            'tab_account': '👤 账户安全',
            'api_key': 'API Key',
            'api_key_hint': '在 {name} 官网获取 API Key',
            'api_key_masked': '已配置 (点击修改)',
            'test_connection': '测试连接',
            'test_success': '✅ 连接成功',
            'test_failed': '❌ 连接失败: {error}',
            'change_password': '修改密码',
            'new_password': '新密码',
            'confirm_password': '确认密码',
            'password_changed': '密码修改成功',
            'password_mismatch': '两次密码不一致',
        },
        'engine': {
            'select': '选择引擎',
            'supported_lang': '支持语言: {langs}',
            'not_configured': '⚠️ 未配置，前往设置',
            'go_settings': '前往设置',
        },
        'upload': {
            'drag': '拖拽文件到此处或点击上传',
            'progress': '上传中 {percent}% ({loaded}/{total})',
            'done': '上传完成',
            'failed': '上传失败',
        },
        'language': {
            'switch': '切换语言',
            'zh': '中文',
            'en': 'English',
        },
    },

    # ==================== 英文 ====================
    'en': {
        'app': {
            'title': 'Subtitle Tool',
            'subtitle': 'Multi-engine ASR for audio/video to subtitle',
        },
        'nav': {
            'home': 'Home',
            'settings': '⚙️ Settings',
            'login': 'Login',
            'logout': 'Logout',
        },
        'common': {
            'submit': 'Submit',
            'cancel': 'Cancel',
            'save': 'Save',
            'delete': 'Delete',
            'edit': 'Edit',
            'loading': 'Loading...',
            'success': 'Success',
            'error': 'Error',
            'confirm_delete': 'Are you sure?',
            'no_data': 'No data',
            'upload': 'Upload File',
        },
        'settings': {
            'title': 'Settings',
            'tab_api': '🔑 API Config',
            'tab_account': '👤 Account',
            'api_key': 'API Key',
            'api_key_hint': 'Get API Key from {name} website',
            'api_key_masked': 'Configured (click to edit)',
            'test_connection': 'Test Connection',
            'test_success': '✅ Connection OK',
            'test_failed': '❌ Failed: {error}',
            'change_password': 'Change Password',
            'new_password': 'New Password',
            'confirm_password': 'Confirm Password',
            'password_changed': 'Password changed',
            'password_mismatch': 'Passwords do not match',
        },
        'engine': {
            'select': 'Select Engine',
            'supported_lang': 'Languages: {langs}',
            'not_configured': '⚠️ Not configured, go to settings',
            'go_settings': 'Go to Settings',
        },
        'upload': {
            'drag': 'Drag files here or click to upload',
            'progress': 'Uploading {percent}% ({loaded}/{total})',
            'done': 'Upload complete',
            'failed': 'Upload failed',
        },
        'language': {
            'switch': 'Switch Language',
            'zh': '中文',
            'en': 'English',
        },
    },

    # ==================== 新增语言模板 ====================
    # 复制上面的 'en' 块，修改 lang_code 和翻译文本即可
    # 'ja': {
    #     'app': {'title': '字幕変換ツール', ...},
    #     ...
    # },
}
```

### 2.3 app.py 集成

```python
from flask import Flask, session, request, jsonify, make_response
from i18n import init_i18n, get_locale, set_language, SUPPORTED_LANGUAGES

app = Flask(__name__)
app.secret_key = 'your-secret-key'

# 初始化 i18n
init_i18n(app)


@app.route('/api/language', methods=['GET'])
def get_language():
    """获取当前语言和支持的语言列表"""
    return jsonify({
        'current': get_locale(),
        'supported': SUPPORTED_LANGUAGES,
    })


@app.route('/api/language', methods=['POST'])
def change_language():
    """切换语言 — 前端语言切换器调用"""
    data = request.get_json()
    lang = data.get('language')
    if set_language(lang):
        resp = make_response(jsonify({'success': True, 'language': lang}))
        resp.set_cookie('language', lang, max_age=365*24*3600)  # 持久化 cookie 1年
        return resp
    return jsonify({'success': False, 'error': 'Unsupported language'}), 400


# Jinja2 模板中直接用 t() 函数
# {{ t('app.title') }}  →  "字幕转文字工具" / "Subtitle Tool"
```

---

## 三、前端实现

### 3.1 templates/base.html — 语言切换器 + JS

```html
<!DOCTYPE html>
<html lang="{{ current_language() }}">
<head>
    <meta charset="UTF-8">
    <title>{{ t('app.title') }}</title>
    <!-- 主题、样式等 -->
</head>
<body>
    <!-- 导航栏 -->
    <nav>
        <a href="/">{{ t('nav.home') }}</a>
        <button onclick="showSettings()">{{ t('nav.settings') }}</button>

        <!-- 🌐 语言切换器 -->
        <div class="lang-switcher">
            {% for code, name in supported_languages().items() %}
            <button class="lang-btn {% if code == current_language() %}active{% endif %}"
                    onclick="switchLanguage('{{ code }}')">
                {{ name }}
            </button>
            {% endfor %}
        </div>

        <a href="https://github.com/yyzq-cf/xxx" target="_blank">GitHub</a>
    </nav>

    {% block content %}{% endblock %}

    <!-- i18n JS -->
    <script>
    // 把后端翻译传给前端 JS
    window.I18N_LANG = '{{ current_language() }}';
    window.SUPPORTED_LANGS = {{ supported_languages() | tojson }};
    </script>
    <script src="{{ url_for('static', filename='js/i18n.js') }}"></script>
</body>
</html>
```

### 3.2 static/js/i18n.js — 前端翻译

```javascript
/**
 * 前端 i18n 翻译模块
 *
 * 用于 JS 动态生成的文本（如 toast、模态框内容等）。
 * 静态 HTML 文本用 Jinja2 的 t() filter，动态 JS 文本用 i18n.t()。
 *
 * 用法:
 *   i18n.t('common.success')           → "操作成功"
 *   i18n.t('upload.progress', {percent: 45, loaded: '12MB', total: '27MB'})
 *                                       → "上传中 45% (12MB/27MB)"
 */

// 前端翻译字典（从后端注入或独立维护）
const I18N_TRANSLATIONS = {
    zh: {
        'common.success': '操作成功',
        'common.error': '操作失败',
        'common.loading': '加载中...',
        'common.confirm_delete': '确认删除？',
        'upload.progress': '上传中 {percent}% ({loaded}/{total})',
        'upload.done': '上传完成',
        'upload.failed': '上传失败',
        'settings.test_success': '✅ 连接成功',
        'settings.test_failed': '❌ 连接失败: {error}',
    },
    en: {
        'common.success': 'Success',
        'common.error': 'Error',
        'common.loading': 'Loading...',
        'common.confirm_delete': 'Are you sure?',
        'upload.progress': 'Uploading {percent}% ({loaded}/{total})',
        'upload.done': 'Upload complete',
        'upload.failed': 'Upload failed',
        'settings.test_success': '✅ Connection OK',
        'settings.test_failed': '❌ Failed: {error}',
    },
};

const i18n = {
    lang: window.I18N_LANG || 'zh',
    fallback: 'en',

    t(key, params = {}) {
        const dict = I18N_TRANSLATIONS[this.lang] || {};
        let text = dict[key];

        // fallback to English
        if (text === undefined && this.lang !== this.fallback) {
            text = (I18N_TRANSLATIONS[this.fallback] || {})[key];
        }
        // fallback to key itself
        if (text === undefined) {
            return key;
        }

        // 替换 {name} 占位符
        for (const [k, v] of Object.entries(params)) {
            text = text.replace(`{${k}}`, v);
        }
        return text;
    },

    /** 动态切换前端语言（配合后端 API） */
    async switch(lang) {
        try {
            const resp = await fetch('/api/language', {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({ language: lang }),
            });
            if (resp.ok) {
                this.lang = lang;
                // 刷新页面让后端 t() filter 也更新
                window.location.reload();
            }
        } catch (e) {
            console.error('Language switch failed:', e);
        }
    },
};

// 语言切换按钮的全局函数（base.html 中 onclick 调用）
function switchLanguage(lang) {
    i18n.switch(lang);
}
```

### 3.3 语言切换器 CSS

```css
/* 语言切换器 */
.lang-switcher {
    display: inline-flex;
    gap: 2px;
    border-radius: 8px;
    overflow: hidden;
    border: 1px solid var(--border);
}

.lang-btn {
    padding: 4px 12px;
    border: none;
    background: var(--bg);
    color: var(--muted);
    cursor: pointer;
    font-size: 0.85rem;
    transition: all 0.2s;
}

.lang-btn:hover {
    background: var(--card);
    color: var(--text);
}

.lang-btn.active {
    background: var(--accent);
    color: #fff;
    font-weight: 600;
}
```

---

## 四、HTML 模板用法

所有模板继承 `base.html`，用 `t()` filter 翻译：

```html
<!-- templates/index.html -->
{% extends 'base.html' %}

{% block content %}
<div class="container">
    <h1>{{ t('app.title') }}</h1>
    <p>{{ t('app.subtitle') }}</p>

    <!-- 引擎选择 -->
    <label>{{ t('engine.select') }}</label>
    <select id="engineSelect">
        <!-- JS 动态填充 -->
    </select>
    <span id="engineHint"></span>

    <!-- 上传区域 -->
    <div id="uploadArea">
        <p>{{ t('upload.drag') }}</p>
    </div>

    <!-- 按钮 -->
    <button class="btn-primary" onclick="startProcess()">
        {{ t('common.submit') }}
    </button>
</div>

<script>
// JS 中用 i18n.t() 翻译动态内容
function showToast(msgKey, params) {
    alert(i18n.t(msgKey, params));
}

function onUploadProgress(percent, loaded, total) {
    document.getElementById('progressLabel').textContent =
        i18n.t('upload.progress', { percent, loaded, total });
}
</script>
{% endblock %}
```

---

## 五、新增语言步骤

### 5.1 添加新语言（3步）

```
1. 在 i18n/__init__.py 的 SUPPORTED_LANGUAGES 中添加:
   'ja': '日本語',

2. 在 i18n/translations.py 中复制 'en' 块，改为:
   'ja': {
       'app': {'title': '字幕変換ツール', ...},
       'nav': {'home': 'ホーム', ...},
       ...
   }

3. (前端) 在 static/js/i18n.js 的 I18N_TRANSLATIONS 中添加对应的 'ja' 块
```

完成。语言切换器自动显示新语言按钮，无需改其他代码。

### 5.2 使用 locales/ JSON 文件（大量翻译时）

```
1. 创建 i18n/locales/ja.json:
   {
     "app": { "title": "字幕変換ツール" },
     "nav": { "home": "ホーム" }
   }

2. 在 __init__.py 中取消 SUPPORTED_LANGUAGES 中 'ja' 的注释

3. init_i18n() 自动加载 locales/*.json，无需改 translations.py
```

---

## 六、翻译 Key 命名规范

| 命名模式 | 示例 | 说明 |
|----------|------|------|
| `namespace.key` | `app.title` | 应用级标题 |
| `nav.xxx` | `nav.settings` | 导航栏元素 |
| `common.xxx` | `common.submit` | 通用按钮/状态 |
| `settings.xxx` | `settings.api_key` | 设置面板 |
| `engine.xxx` | `engine.select` | 引擎选择区 |
| `upload.xxx` | `upload.progress` | 上传相关 |

**规则：**
1. 全小写，用 `.` 分层级
2. namespace 对应功能模块（nav/common/settings/engine/upload）
3. 同一 key 在所有语言中必须存在（缺失时 fallback 英文 → key 本身）
4. 变量用 `{name}` 格式：`t('settings.api_key_hint', name='BCut')`

---

## 七、测试 i18n

### 7.1 自动化测试

```python
import pytest

def test_translation_keys_consistency():
    """所有语言的 key 必须一致"""
    from i18n.translations import TRANSLATIONS

    def get_keys(obj, prefix=''):
        keys = set()
        for k, v in obj.items():
            full = f'{prefix}.{k}' if prefix else k
            if isinstance(v, dict):
                keys |= get_keys(v, full)
            else:
                keys.add(full)
        return keys

    lang_keys = {}
    for lang, data in TRANSLATIONS.items():
        lang_keys[lang] = get_keys(data)

    base = lang_keys['zh']  # 以中文为基准
    for lang, keys in lang_keys.items():
        missing = base - keys
        extra = keys - base
        assert not missing, f'[{lang}] missing keys: {missing}'
        assert not extra, f'[{lang}] extra keys: {extra}'


def test_locale_detection():
    """测试语言检测优先级"""
    with app.test_client() as client:
        # 默认语言
        resp = client.get('/')
        assert resp.status_code == 200

        # 切换语言
        resp = client.post('/api/language', json={'language': 'en'})
        assert resp.status_code == 200
        assert resp.get_json()['language'] == 'en'

        # 不支持的语言
        resp = client.post('/api/language', json={'language': 'xx'})
        assert resp.status_code == 400


def test_translation_fallback():
    """翻译缺失时 fallback 到英文"""
    with app.test_request_context('/'):
        from i18n import t
        # 假设 key 只存在于 en 中
        # t() 应返回英文文本而非 key 本身
        assert t('app.title') is not None
```

### 7.2 手动测试清单

```
□ 首次访问自动检测浏览器语言
□ 点语言切换器 → 页面全部文本更新
□ 刷新页面 → 语言保持（session/cookie 持久化）
□ JS 动态内容（toast/alert/进度条）也跟随语言切换
□ 引擎名称等后端数据不翻译（只翻译 UI 文案）
□ 每种语言的 key 完整，没有显示 key 本身的情况
```

---

## 八、注意事项

1. **引擎名称/服务名不翻译** — "BCut"、"SiliconFlow"、"Groq" 等品牌名在所有语言中保持原文，只翻译描述性文案
2. **API 返回的业务数据不翻译** — ASR 识别结果、翻译结果等由业务逻辑产生，不经过 i18n
3. **`data-i18n` 属性方案（可选）** — 如果项目大量使用静态 HTML，可以在元素上加 `data-i18n="key"` 属性，JS 遍历自动替换。但 Jinja2 `{{ t('key') }}` 更直观，推荐用后者
4. **翻译文件不要和 config.json 混** — 翻译是代码的一部分，放在 `i18n/` 目录；用户配置放 `data/` 目录
5. **Docker 镜像包含所有语言文件** — `i18n/` 目录要 COPY 进镜像（不是敏感数据，不需要 .dockerignore 排除）
6. **新增语言不改 CI/CD** — 翻译文件随代码 push 自动构建进新镜像，无额外配置
