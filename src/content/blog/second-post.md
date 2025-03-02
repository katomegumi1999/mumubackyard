---
title: 'AI批量提取图片信息'
description: 'python调用阿里云百炼api'
pubDate: '2025-02-01'
heroImage: '/blog-placeholder-2.jpg'
---

通过python调用阿里云api返回生成信息txt
```
import os
import time
from datetime import datetime
from dashscope import MultiModalConversation

def process_images(folder_path):
    # 支持的图片格式列表
    image_extensions = ['.png', '.jpg', '.jpeg', '.bmp', '.gif', '.webp']
    
    # 获取文件列表并过滤
    all_files = os.listdir(folder_path)
    image_files = [f for f in all_files 
                  if os.path.splitext(f)[1].lower() in image_extensions]
    total_files = len(image_files)
    
    print(f"📁 开始处理文件夹：{folder_path}")
    print(f"🔍 发现 {len(all_files)} 个文件，其中 {total_files} 个图片文件")
    print("─" * 50)

    # 初始化统计信息
    processed = 0
    success_count = 0
    error_count = 0
    start_time = time.time()

    for idx, filename in enumerate(image_files, 1):
        file_start = time.time()
        file_path = os.path.join(folder_path, filename)
        
        # 打印文件头信息
        print(f"\n📄 正在处理文件 ({idx}/{total_files}):")
        print(f"  文件名：{filename}")
        print(f"  文件大小：{os.path.getsize(file_path)/1024:.2f} KB")
        print(f"  开始时间：{datetime.now().strftime('%H:%M:%S')}")

        try:
            # 构建请求
            messages = [{
                "role": "user",
                "content": [
                    {"image": f"file://{file_path}"},
                    {"text": "这里填入想要发送给ai的文字" }
                ]
            }]

            # API调用
            print("🔄 正在调用Qwen-VL API...")
            api_start = time.time()
            response = MultiModalConversation.call(
                model='qwen-vl-max-latest',
                messages=messages
            )
            api_time = time.time() - api_start

            # 处理响应
            if response.status_code == 200:
                result_text = response.output.choices[0].message.content[0]["text"]
                status = "✅ 成功"
                success_count += 1
            else:
                result_text = f"Error: {response.code} - {response.message}"
                status = "⚠️ API错误"
                error_count += 1

            # 保存结果
            output_path = os.path.join(folder_path, 
                                      os.path.splitext(filename)[0] + ".txt")
            with open(output_path, 'w', encoding='utf-8') as f:
                f.write(result_text)

            # 计算进度
            processed += 1
            elapsed = time.time() - start_time
            avg_time = elapsed / processed if processed > 0 else 0
            remaining = avg_time * (total_files - processed)

            # 打印结果摘要
            print(f"\n{status}")
            print(f"  API响应时间：{api_time:.2f}s")
            print(f"  结果长度：{len(result_text)} 字符")
            print(f"  保存路径：{output_path}")
            
        except Exception as e:
            error_count += 1
            print(f"\n❌ 处理失败：{str(e)}")
            print(f"  错误类型：{type(e).__name__}")
            if hasattr(e, 'errno'):
                print(f"  错误代码：{e.errno}")

        finally:
            # 打印文件统计
            file_time = time.time() - file_start
            print(f"\n⏱️ 本文件耗时：{file_time:.2f}s")
            print(f"📊 进度：{processed}/{total_files} "
                 f"({processed/total_files*100:.1f}%)")
            print(f"⏳ 剩余时间估计：{remaining/60:.1f} 分钟")
            print("─" * 50)

    # 最终统计报告
    total_time = time.time() - start_time
    print("\n🎉 处理完成！")
    print(f"总耗时：{total_time/60:.2f} 分钟")
    print(f"成功：{success_count} 文件")
    print(f"失败：{error_count} 文件")
    print(f"平均处理速度：{total_time/total_files:.2f}s/文件")

if __name__ == "__main__":
    image_folder = r"C:\arcgisfile\testpic"
    
    if os.path.exists(image_folder) and os.path.isdir(image_folder):
        try:
            process_images(image_folder)
        except KeyboardInterrupt:
            print("\n🛑 用户中断操作！")
    else:
        print("错误：文件夹路径无效或不存在")```
