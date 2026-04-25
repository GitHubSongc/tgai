import base64
import html
import json
import logging
import os
import re

import requests
import telebot
from ddgs import DDGS

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(message)s",
)
log = logging.getLogger("bot")

BOT_TOKEN = os.getenv("BOT_TOKEN")
DEEPSEEK_API_KEY = os.getenv("sk-ed60f511ae254ac3999ba96454cef63d")
DEEPSEEK_API_URL = "https://api.deepseek.com/v1/chat/completions"
DEEPSEEK_MODEL = "deepseek-v4-pro"  # DeepSeek V4 Pro — locked
DEEPSEEK_MAX_OUTPUT_TOKENS = 10000
SYSTEM_PROMPT = (
    "你是一个乐于助人的中文助理，回答简洁、准确、友好。\n"
    "\n"
    "【联网搜索】当用户询问最新事件、实时数据、或你不确定的事实时，"
    "请调用 web_search 工具联网搜索后再作答，并在回答中标注信息来源。\n"
    "\n"
    "【图片】用户发送图片时，你会先收到一段以「[图片描述]」开头的视觉模型生成的描述，"
    "请基于该描述结合用户问题作答。\n"
    "\n"
    "【公式输出 — 严格执行】所有数学、物理公式必须使用 Unicode 字符直接渲染，"
    "严禁使用 LaTeX（不要出现 $...$、\\frac、\\int、\\vec、\\hat、\\nabla、\\langle 等任何反斜杠命令），"
    "也严禁用纯文字伪代码（不要写 sqrt、^、_、x^2、a/b 这类）。\n"
    "请直接使用以下 Unicode 字符：\n"
    "- 上标：⁰¹²³⁴⁵⁶⁷⁸⁹⁺⁻⁼⁽⁾ⁿⁱ；下标：₀₁₂₃₄₅₆₇₈₉₊₋₌₍₎ₐₑₒₓᵢⱼₖₗₘₙₚₛₜ\n"
    "- 希腊字母：αβγδεζηθικλμνξοπρστυφχψω ΑΒΓΔΕΖΗΘΙΚΛΜΝΞΟΠΡΣΤΥΦΧΨΩ\n"
    "- 微积分：∫ ∬ ∭ ∮ ∂ d/dx（用斜线即可）、极限用 lim\n"
    "- 矢量：用粗体或加上箭头符号 →，如 𝐄、𝐁 或 E⃗、B⃗；点乘 · 叉乘 ×\n"
    "- 偏微分：∂；梯度 ∇；散度 ∇·；旋度 ∇×；拉普拉斯 ∇²；δ ε\n"
    "- 求和/求积：∑ ∏；根号 √ ∛ ∜；正负 ± ∓；无穷 ∞\n"
    "- 关系：≤ ≥ ≠ ≈ ≡ ≅ ∼ ∝ → ↔ ⇒ ⇔ ⊂ ⊆ ⊃ ⊇ ∈ ∉ ∪ ∩ ∅\n"
    "- 量子力学：ℏ、|ψ⟩、⟨φ|、⟨φ|ψ⟩、Ĥ（用组合字符 H + ̂ ）、算符上加帽用 ̂ \n"
    "- 电磁学示例：∇×𝐄 = −∂𝐁/∂t、∇·𝐄 = ρ/ε₀\n"
    "- 矩阵用方括号或圆括号配换行手工排版，例如：\n"
    "    ⎡ 1  2 ⎤\n"
    "    ⎣ 3  4 ⎦\n"
    "  或 ((1, 2), (3, 4))\n"
    "示例输出：E = mc²、∫₀^∞ e⁻ˣ² dx = √π/2、Ĥ|ψ⟩ = E|ψ⟩\n"
    "（如果某个特殊符号没有对应 Unicode，可以用最接近的 Unicode 表达，但绝不允许出现反斜杠 LaTeX。）\n"
    "\n"
    "【加粗】需要加粗时直接使用 **加粗内容**，会被自动转换为 Telegram 原生粗体，"
    "不要使用其它强调符号。"
