---
title: 'cad文件dwg自动批量转shp'
description: 'dwg自动批量转shp'
pubDate: '2025-02-01'
heroImage: '/blog-placeholder-2.jpg'
---

通过构建arcgispro工具箱脚本实现dwg自动转shp，支持id选择，路径选择
```
import arcpy
import os
import re

def get_unique_shp_path(output_shp):
    """
    检查指定路径的.shp文件是否存在，如果存在则添加后缀生成唯一文件名。
    """
    base, ext = os.path.splitext(output_shp)
    counter = 1
    while os.path.exists(output_shp):
        output_shp = f"{base}_{counter}{ext}"
        counter += 1
    return output_shp

def sanitize_filename(filename):
    """
    将文件名中的特殊字符替换为下划线，仅保留中文字符、字母和数字。
    """
    # 替换所有非中文、字母、数字字符为下划线
    sanitized_filename = re.sub(r'[^\w\u4e00-\u9fa5]+', '_', filename)
    return sanitized_filename

def process_dwg_files(dwg_files, feature_limit, oids=None):
    for dwg_path in dwg_files:
        dwg_path = os.path.abspath(dwg_path.strip("'\""))
        arcpy.AddMessage(f"正在处理文件: {dwg_path}")
        
        # 检查文件是否为DWG格式
        if not dwg_path.lower().endswith(".dwg"):
            arcpy.AddWarning(f"警告: 文件 {dwg_path} 不是DWG格式，跳过。")
            continue
        
        # 设置工作空间为DWG文件
        arcpy.env.workspace = dwg_path
        
        # 获取DWG文件的名称（不带扩展名）
        dwg_name = os.path.splitext(os.path.basename(dwg_path))[0]
        
        # 对文件名进行清理，去掉特殊字符
        sanitized_dwgn = sanitize_filename(dwg_name)
        
        # 查找名为 polygon 的 CAD 要素类
        polygon_fc = "polygon"
        
        # 检查 polygon 要素类是否存在
        if not arcpy.Exists(polygon_fc):
            arcpy.AddWarning(f"警告: 文件 {dwg_name} 中未找到名为 'polygon' 的要素类，跳过。")
            continue
        
        try:
            # 获取属性表中有效 OID 的要素数量（确保仅统计有效记录）
            query = "OID IS NOT NULL"  # 确保 OID 不为空
            arcpy.management.MakeFeatureLayer(polygon_fc, "polygon_layer")
            arcpy.management.SelectLayerByAttribute("polygon_layer", "NEW_SELECTION", query)
            
            # 获取选择的有效要素数量
            feature_count = int(arcpy.management.GetCount("polygon_layer")[0])
            arcpy.AddMessage(f"属性表中的有效要素数量: {feature_count}")
            
            # 如果提供了 OID 参数，构建查询条件以选择特定 OID
            if oids:
                oids_query = f"OID IN ({','.join(map(str, oids))})"
                arcpy.management.SelectLayerByAttribute("polygon_layer", "NEW_SELECTION", oids_query)
                arcpy.AddMessage(f"选择了指定的 OID: {oids_query}")
            else:
                # 如果没有提供 OID，则使用 feature_limit 来限制选择的要素数量
                if feature_count >= feature_limit:
                    arcpy.AddWarning(f"警告: 文件 {dwg_name} 中的有效要素数量大于等于 {feature_limit}，跳过。")
                    continue
                arcpy.management.SelectLayerByAttribute("polygon_layer", "CLEAR_SELECTION")

            # 导出为SHP文件
            if output_path:
                # 使用用户指定的输出路径
                output_shp = os.path.join(output_path, f"{sanitized_dwgn}.shp")
            else:
                # 使用默认路径
                output_folder = os.path.dirname(dwg_path)
                output_shp = os.path.join(output_folder, f"{sanitized_dwgn}.shp")
            
            # 检查文件是否存在，若存在则修改文件名
            output_shp = get_unique_shp_path(output_shp)
            
            # 导出选择的要素
            arcpy.management.CopyFeatures("polygon_layer", output_shp)
            arcpy.AddMessage(f"导出成功: {output_shp}")
        
        except Exception as e:
            arcpy.AddError(f"处理文件 {dwg_name} 时出错: {e}")
            continue

if __name__ == "__main__":
    # 假设你手动输入了DWG文件路径
    dwg_files = arcpy.GetParameterAsText(0).split(";")

    # 获取FeatureLimit参数，如果为空则默认为10
    feature_limit = arcpy.GetParameterAsText(1)
    feature_limit = int(feature_limit) if feature_limit else 10  # 默认10
    
    # 获取OID参数，如果有值则使用逗号分隔的值，否则为空
    oids_input = arcpy.GetParameterAsText(2)
    oids = [int(oid.strip()) for oid in oids_input.split(",")] if oids_input else None

    # 获取输出路径
    output_path = arcpy.GetParameterAsText(3)  # 假设第二个参数是输出路径
    if output_path:
        output_path = os.path.abspath(output_path.strip("'\""))
    else:
        output_path = None  # 如果未提供输出路径，保持默认
    
    # 确保每个文件路径都经过特殊字符处理，使用os.path.abspath
    dwg_files = [os.path.abspath(dwg.strip("'\"")) for dwg in dwg_files]
    
    # 处理DWG文件
    process_dwg_files(dwg_files, feature_limit, oids)
```