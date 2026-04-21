# Dataset Rules - Chemistry

本文档记录了化学数据集的处理规则和脚本。

## 1. 数据提取

从 `China-K12-STEM-10K-CoT-Reasoning` 中提取化学题目并转换为 Markdown 格式。

### 数据源

- `train.jsonl` - 训练集
- `test.jsonl` - 测试集
- `validation.jsonl` - 验证集

### 输出格式

每个题目一个 Markdown 文件，命名格式：`Chemistry_Q{序号:03d}_{哈希}.md`

### 提取脚本

```python
import json
import os
import hashlib

output_dir = "dt_k12_stem_chemistry"
os.makedirs(output_dir, exist_ok=True)

data_files = [
    "China-K12-STEM-10K-CoT-Reasoning/train.jsonl",
    "China-K12-STEM-10K-CoT-Reasoning/test.jsonl",
    "China-K12-STEM-10K-CoT-Reasoning/validation.jsonl"
]

chemistry_count = 0

for data_file in data_files:
    with open(data_file, 'r', encoding='utf-8') as f:
        for line in f:
            if not line.strip():
                continue

            data = json.loads(line)

            # 筛选化学题目
            if data.get('subject') != 'Chemistry':
                continue

            chemistry_count += 1

            # 生成哈希ID
            question_id = data.get('question_id', '')
            hash_obj = hashlib.md5(question_id.encode())
            hash_hex = hash_obj.hexdigest()[:8]

            # 构造文件名
            filename = f"Chemistry_Q{chemistry_count:03d}_{hash_hex}.md"
            filepath = os.path.join(output_dir, filename)

            # 提取数据
            question_text = data.get('question_text', '').replace('\\(', '$').replace('\\)', '$').replace('\\[', '$$').replace('\\]', '$$')
            solution_steps = data.get('solution_steps', [])
            knowledge_points = data.get('knowledge_points', [])
            knowledge_points_en = data.get('knowledge_points_en', [])
            grade = data.get('grade', '')
            grade_level = data.get('grade_level', f'Grade {grade}')
            difficulty = data.get('difficulty', '')
            question_type = data.get('question_type', '')

            # 格式化解答步骤
            solution_text = ""
            for i, step in enumerate(solution_steps, 1):
                step_clean = step.replace('\\(', '$').replace('\\)', '$').replace('\\[', '$$').replace('\\]', '$$')
                solution_text += str(i) + ". " + step_clean + "\n"

            # 格式化知识点
            kp_text = "**中文**:\n"
            for kp in knowledge_points:
                kp_text += "  - " + kp + "\n"
            kp_text += "\n**English**:\n"
            for kp_en in knowledge_points_en:
                kp_text += "  - " + kp_en + "\n"

            # 写入 Markdown 文件
            header = "# Chemistry - 题目 " + str(chemistry_count) + "\n\n"
            meta = "**ID**: " + question_id + "\n"
            meta += "**年级**: " + grade_level + "\n"
            meta += "**难度**: " + difficulty + "\n"
            meta += "**题型**: " + question_type + "\n\n"

            q_section = "## 题目\n" + question_text + "\n\n"
            a_section = "## 解答\n\n" + solution_text + "\n"
            k_section = "## 知识点\n\n" + kp_text + "\n---\n"

            md_content = header + meta + q_section + a_section + k_section

            with open(filepath, 'w', encoding='utf-8') as out_f:
                out_f.write(md_content)

            if chemistry_count % 200 == 0:
                print(f"已提取 {chemistry_count} 道化学题...")

print(f"完成! 共提取 {chemistry_count} 道化学题目")
```

## 2. 图片下载

### 代理设置

图片托管服务器需要代理访问：
```bash
export HTTP_PROXY=http://127.0.0.1:1087
export HTTPS_PROXY=http://127.0.0.1:1087
```

### 下载脚本

```python
import json
import os
import hashlib
import urllib.request

output_dir = "dt_k12_stem_chemistry"
images_dir = os.path.join(output_dir, "images")
os.makedirs(images_dir, exist_ok=True)

data_files = [
    "China-K12-STEM-10K-CoT-Reasoning/train.jsonl",
    "China-K12-STEM-10K-CoT-Reasoning/test.jsonl",
    "China-K12-STEM-10K-CoT-Reasoning/validation.jsonl"
]

chemistry_count = 0
image_success = 0
image_fail = 0

def download_image(url, local_path):
    """下载图片，需要添加请求头避免403错误"""
    global image_success, image_fail
    try:
        if os.path.exists(local_path):
            image_success += 1
            return True
        req = urllib.request.Request(url, headers={
            'User-Agent': 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36',
            'Referer': 'https://huggingface.co/'
        })
        with urllib.request.urlopen(req, timeout=30) as response:
            with open(local_path, 'wb') as f:
                f.write(response.read())
        image_success += 1
        return True
    except Exception as e:
        image_fail += 1
        return False

for data_file in data_files:
    with open(data_file, 'r', encoding='utf-8') as f:
        for line in f:
            if not line.strip():
                continue

            data = json.loads(line)

            if data.get('subject') != 'Chemistry':
                continue

            chemistry_count += 1
            question_images = data.get('question_images', [])

            for img_url in question_images:
                img_name = img_url.split('/')[-1]
                local_img_name = f"{chemistry_count:03d}_{img_name}"
                local_img_path = os.path.join(images_dir, local_img_name)
                download_image(img_url, local_img_path)

            if chemistry_count % 200 == 0:
                print(f"已处理 {chemistry_count} 道题... (已下载图片: {image_success}, 失败: {image_fail})")

print(f"\n完成! 共处理 {chemistry_count} 道化学题")
print(f"图片下载成功: {image_success} 张")
print(f"图片下载失败: {image_fail} 张")
```

### 添加图片引用到 Markdown

```python
import json
import os
import hashlib

output_dir = "dt_k12_stem_chemistry"
images_dir = os.path.join(output_dir, "images")

data_files = [
    "China-K12-STEM-10K-CoT-Reasoning/train.jsonl",
    "China-K12-STEM-10K-CoT-Reasoning/test.jsonl",
    "China-K12-STEM-10K-CoT-Reasoning/validation.jsonl"
]

chemistry_count = 0
updated_count = 0

for data_file in data_files:
    with open(data_file, 'r', encoding='utf-8') as f:
        for line in f:
            if not line.strip():
                continue

            data = json.loads(line)
            if data.get('subject') != 'Chemistry':
                continue

            chemistry_count += 1
            question_id = data.get('question_id', '')
            hash_obj = hashlib.md5(question_id.encode())
            hash_hex = hash_obj.hexdigest()[:8]

            filename = f"Chemistry_Q{chemistry_count:03d}_{hash_hex}.md"
            filepath = os.path.join(output_dir, filename)

            if not os.path.exists(filepath):
                continue

            with open(filepath, 'r', encoding='utf-8') as f:
                content = f.read()

            # 如果已经有图片引用，跳过
            if '![' in content:
                continue

            question_images = data.get('question_images', [])
            image_refs = []
            for img_url in question_images:
                img_name = img_url.split('/')[-1]
                local_img_name = f"{chemistry_count:03d}_{img_name}"
                local_img_path = os.path.join(images_dir, local_img_name)

                if os.path.exists(local_img_path):
                    image_refs.append(f"images/{local_img_name}")

            if image_refs:
                question_text = data.get('question_text', '').replace('\\(', '$').replace('\\)', '$').replace('\\[', '$$').replace('\\]', '$$')
                new_question_section = "## 题目\n" + question_text + "\n\n"
                for img_ref in image_refs:
                    new_question_section += f"![{img_ref}]({img_ref})\n"
                new_question_section += "\n"

                content = content.replace("## 题目\n" + question_text + "\n\n", new_question_section)

                with open(filepath, 'w', encoding='utf-8') as f:
                    f.write(content)
                updated_count += 1

print(f"完成! 已更新 {updated_count} 个文件")
```

## 3. 文件重命名

### 命名规则

从 `Chemistry_Q001_xxxxxxxx.md` 重命名为 `grade{年级}_{难度}_{题型}_{ID前7位}.md`

示例：`grade9_medium_inference_4a5cf47.md`

### 题型翻译规则

| 中文题型 | 英文翻译 |
|---------|---------|
| 选择题 | multiple_choice |
| 单选题 | single_choice |
| 多选题/双选题/多项选择题 | multiple_select |
| 填空题 | fill_in_the_blank |
| 填空与简答 | fill_and_short_answer |
| 计算题 | calculation |
| 解答题 | solution |
| 简答题 | short_answer |
| 实验题/实验探究题 | experiment |
| 实验探究 | experiment_inquiry |
| 实验设计 | experiment_design |
| 推断题 | inference |
| 有机推断题 | organic_inference |
| 无机推断题 | inorganic_inference |
| 综合题 | comprehensive |
| 工业流程题 | industrial_process |
| 结构与性质 | structure_property |
| 证明题 | proof |
| 作图题 | drawing |
| 达标练 | practice |
| 基础练/基础过关练 | basic_practice |
| 刷真题 | exam_review |
| 课时作业/课时对点练 | classwork |
| 课后巩固练习/课后习题 | homework |
| 复习与提高 | review_advanced |
| 强化训练 | intensive_training |
| 刷提升 | review_advanced |
| 刷能力 | ability_test |
| 刷基础 | review_basic |
| 刷速度 | speed_drill |
| 刷易错 | error_prone_drill |
| 能力提升/能力提升练 | ability_improvement |
| 易混易错练 | confusion_prone_drill |
| 专题强化练 | topic_intensive |
| 三年模拟练 | three_year_simulation |
| 思想方法练 | method_training |
| 课内填空 | in_class_fill |
| 科学阅读 | science_reading |
| 科学探究 | scientific_inquiry |
| 流程图 | flowchart |
| 未知 | unknown |
| 自我评价与作业 | self_assessment |

### 重命名脚本

```python
import os
import re

def normalize_type(qtype):
    """将题型规范化为英文（化学专业类型）"""
    qtype = qtype.strip()

    # 已经是英文的标准类型
    english_types = {
        'Calculation': 'calculation',
        'Multiple-choice': 'multiple_choice',
        'Fill-in-the-blank': 'fill_in_the_blank',
        'Multiple-select': 'multiple_select',
        'Experiment': 'experiment',
        'Drawing': 'drawing',
        'Comprehensive': 'comprehensive',
    }
    if qtype in english_types:
        return english_types[qtype]

    # 包含证明或演算步骤的描述
    if any(kw in qtype for kw in ['证明过程', '演算步骤', '推理过程', '解答题']):
        return 'calculation'

    # 中文类型翻译
    translations = [
        ('单选题', 'single_choice'), ('多选题', 'multiple_select'),
        ('双选题', 'multiple_select'), ('多项选择题', 'multiple_select'),
        ('选择题', 'multiple_choice'), ('填空题', 'fill_in_the_blank'),
        ('填空与简答', 'fill_and_short_answer'), ('填空', 'fill'),
        ('双空题', 'double_blank'), ('实验题', 'experiment'),
        ('实验探究题', 'experiment'), ('实验探究', 'experiment_inquiry'),
        ('实验设计', 'experiment_design'), ('计算题', 'calculation'),
        ('解答题', 'solution'), ('作图题', 'drawing'),
        ('证明题', 'proof'), ('简答题', 'short_answer'),
        ('简答', 'short_answer'), ('推断题', 'inference'),
        ('有机推断题', 'organic_inference'), ('无机推断题', 'inorganic_inference'),
        ('物质推断', 'inference'), ('结构推断', 'structure_inference'),
        ('合成推断', 'synthesis_inference'), ('推断', 'inference'),
        ('综合题', 'comprehensive'),
    ]
    for cn, en in translations:
        if cn in qtype:
            return en

    # 化学特殊类型
    special_mappings = {
        '工业流程题': 'industrial_process',
        '结构与性质': 'structure_property',
        '纯碱生产历史与侯氏制碱法': 'soda_production',
        '化学平衡状态的判断': 'chemical_equilibrium',
        '钠及其氧化物': 'sodium_oxide',
        '常见化学电源': 'chemical_power',
        '羧酸衍生物': 'carboxylic_derivatives',
        '元素的性质': 'element_properties',
        '流程图': 'flowchart',
        '科学探究': 'scientific_inquiry',
        '科学阅读': 'science_reading',
    }
    for cn, en in special_mappings.items():
        if cn in qtype:
            return en

    # 练习/作业类型
    if '达标练' in qtype:
        return 'practice'
    elif '基础练' in qtype or '基础过关练' in qtype:
        return 'basic_practice'
    elif '刷真题' in qtype:
        return 'exam_review'
    elif '课时作业' in qtype or '课时对点练' in qtype:
        return 'classwork'
    elif '课后巩固练习' in qtype or '课后习题' in qtype or '巩固提升' in qtype:
        return 'homework'
    elif '复习与提高' in qtype:
        return 'review_advanced'
    elif '自我评价与作业' in qtype:
        return 'self_assessment'
    elif '根据要求作答' in qtype:
        return 'short_answer'
    elif '未知' in qtype:
        return 'unknown'
    elif '综合' in qtype:
        return 'comprehensive'

    # 训练/刷题类型
    if '强化训练' in qtype:
        return 'intensive_training'
    elif '刷提升' in qtype:
        return 'review_advanced'
    elif '刷能力' in qtype:
        return 'ability_test'
    elif '刷基础' in qtype:
        return 'review_basic'
    elif '刷速度' in qtype:
        return 'speed_drill'
    elif '刷易错' in qtype:
        return 'error_prone_drill'
    elif '能力提升' in qtype or '能力提升练' in qtype:
        return 'ability_improvement'
    elif '易混易错练' in qtype:
        return 'confusion_prone_drill'
    elif '专题强化练' in qtype:
        return 'topic_intensive'
    elif '三年模拟练' in qtype:
        return 'three_year_simulation'
    elif '思想方法练' in qtype:
        return 'method_training'
    elif '课内填空' in qtype:
        return 'in_class_fill'

    # 处理包含分数的描述
    if '分' in qtype and ('题' in qtype or '本大题' in qtype):
        return 'calculation'

    # 默认清理
    result = re.sub(r'[^\w_]', '_', qtype.lower())[:30]
    return result if result else 'other'

def parse_and_rename(filepath):
    """解析并重命名单个文件"""
    with open(filepath, 'r', encoding='utf-8') as f:
        content = f.read()

    # 提取ID
    id_match = re.search(r'ID.*?[:：]\s*([a-f0-9]+)', content)
    if not id_match:
        return None, None
    question_id = id_match.group(1)[:7]

    # 提取年级
    grade_match = re.search(r'年级.*?[:：]\s*Grade (\d+)', content)
    if not grade_match:
        return None, None
    grade = f"grade{grade_match.group(1)}"

    # 提取难度
    difficulty_match = re.search(r'难度.*?[:：]\s*(.+)', content)
    if not difficulty_match:
        return None, None
    difficulty = difficulty_match.group(1).strip().lower().replace('-', '_').replace(' ', '_')

    # 提取题型
    type_match = re.search(r'题型.*?[:：]\s*(.+)', content)
    if not type_match:
        return None, None
    qtype_raw = type_match.group(1).strip()
    qtype = normalize_type(qtype_raw)

    new_name = f"{grade}_{difficulty}_{qtype}_{question_id}.md"
    return filepath, new_name

# 处理所有文件
md_files = sorted([f for f in os.listdir('.') if f.endswith('.md')])
renamed = 0
skipped = 0

for old_name in md_files:
    old_path, new_name = parse_and_rename(old_name)
    if not old_path or old_name == new_name or os.path.exists(new_name):
        skipped += 1
        continue
    try:
        os.rename(old_name, new_name)
        renamed += 1
        if renamed % 100 == 0:
            print(f"已重命名 {renamed} 个文件...")
    except Exception as e:
        print(f"错误: {old_name} - {e}")

print(f"完成! 重命名成功: {renamed} 个, 跳过: {skipped} 个")
```

## 4. 处理结果统计

| 项目 | 数量 |
|------|------|
| 化学题目总数 | 3014 |
| 下载图片 | 3167 张 |
| Markdown 文件 | 3014 个 |
| 包含图片的文件 | ~235 个 |

### 年级分布

| 年级 | 数量 |
|------|------|
| Grade 9 | 1376 |
| Grade 12 | 590 |
| Grade 10 | 546 |
| Grade 11 | 471 |
| Grade 8 | 31 |

### 难度分布

| 难度 | 数量 |
|------|------|
| Medium | 2254 |
| Easy | 610 |
| Hard | 150 |

### 题型分布（主要）

| 题型 | 数量 |
|------|------|
| hard (hard_* 系列) | 1350 |
| multiple (multiple_* 系列) | 1140 |
| fill (fill_* 系列) | 189 |
| experiment | 82 |
| scientific (scientific_* 系列) | 65 |
| calculation | 57 |
| comprehensive | 22 |
| short (short_* 系列) | 19 |
| deduction | 15 |
| inference | 9 |
| flowchart | 8 |
| review (review_* 系列) | 8 |
