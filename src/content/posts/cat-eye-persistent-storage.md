---
title: 小米猫眼持久化存储方案
description: 基于Xiaomi Miot + shell脚本自动化获取并存储小米智能猫眼2视频
published: 2026-07-29
draft: false
tags: [智能家居]
toc: true
---

## 背景

小米智能猫眼2的视频是3天滚动存储，想要7/30天的就要收费了，懂的都懂，也可以搞个储存卡，想着家里有NAS，能不能搞到NAS里存储

## Home Assistant

飞牛在应用里先安装Home Assistant，把`config`目录挂载下，方便修改，把我们挂载视频的目录也挂载出来，方便查看
|本地路劲|装载路径|
|--|--|
|/vol1/@appshare/Home-Assistant/config|/config|
|/vol1/1000/小米智能猫眼2|/media/xiaomi_video|

## Xiaomi Miot安装

然后进入HA的管理页面，进入设置->设备与服务->添加集成->搜索HACS->下载，下载完毕后设置页面顶部会提示会需要重启，重启就行，重启后页面左侧菜单栏会有HACS菜单，进入HACS搜索Xiaomi Miot安装，然后同样需要重启+集成，集成时会让你登录小米账号，跟着操作就行

## 脚本

### 第一步

ssh进入飞牛，在`/vol1/@appshare/Home-Assistant/config`新增`xiaomi_video_autosave.sh`

```shell
#!/bin/bash

## 你可以在这里修改设置
path="/media/xiaomi_video/"        # 视频存放路径，注意结尾的 "/"
video_limit=500                     # 视频存储数量上限
log_dir="/tmp/xiaomi_video_log"     # 日志目录
tmp_dir="/tmp/xiaomi_video_tmp"     # 临时分片目录
mkdir -p "$log_dir" "$tmp_dir" "$path"

# 1. 检查视频数量，超出则删除最早的视频
current_file_size=$(ls -1 "$path" 2>/dev/null | wc -l)
video_url="$1"
raw_file_name=$(echo "$2" | sed 's/[^0-9]//g')
file_name="${path}${raw_file_name}.mp4"
log_file="${log_dir}/${raw_file_name}.log"
seg_list="${tmp_dir}/${raw_file_name}_concat.txt"

if [ "$current_file_size" -ge "$video_limit" ]; then
    ls -1tr "$path" | head -n $(($current_file_size - $video_limit + 1)) | while read -r f; do
        rm -f "${path}${f}"
    done
fi

# 清理本次临时文件
rm -f "${tmp_dir}/${raw_file_name}"_* "$seg_list"

# 2. 方案说明（为什么不能直接用 ffmpeg -f hls 一步出 MP4）：
#    小米猫眼的 m3u8 每个分片是独立的完整 MP4（带独立 moov box）。ffmpeg 的 HLS
#    demuxer 在遇到第 2 个分片的 moov 时会报 "Found duplicated MOOV Atom"，导致
#    解码器状态重置、DTS 乱序、demuxing 失败，最终只输出第 1 个分片（约 3-6 秒）。
#    无论是 -c copy、重编码、-f segment 都绕不过这个问题，因为根因在 HLS demuxer
#    层面。
#
#    解决方案：完全绕过 ffmpeg 的 HLS demuxer。
#    1) 用 curl 下载 m3u8 文本
#    2) 解析出 KEY URI、IV 和每个分片 URL
#    3) 为每个分片构造一个"只含 1 个分片的独立 m3u8"
#    4) 用 ffmpeg 逐个下载这些单分片 m3u8 → 独立 mp4（每个 ffmpeg 只看到 1 个
#       moov，不会 duplicated）
#    5) 用 concat demuxer 拼接所有独立 mp4

echo "===== Step 1: 下载 m3u8 清单 =====" | tee "$log_file"
m3u8_content=$(curl -s -A "Mozilla/5.0 (Linux; Android 10)" "$video_url")
if [ -z "$m3u8_content" ]; then
    echo "[✗] m3u8 下载失败（空内容），URL 可能已过期" | tee -a "$log_file"
    exit 1
fi

echo "m3u8 内容行数: $(echo "$m3u8_content" | wc -l)" | tee -a "$log_file"

# 3. 解析 KEY URI 和 IV
key_uri=$(echo "$m3u8_content" | grep '#EXT-X-KEY' | sed -n 's/.*URI="\([^"]*\)".*/\1/p')
key_iv=$(echo "$m3u8_content" | grep '#EXT-X-KEY' | sed -n 's/.*IV=\(0x[0-9A-Fa-f]*\).*/\1/p')

if [ -z "$key_uri" ]; then
    echo "[✗] 未能解析出 KEY URI" | tee -a "$log_file"
    exit 1
fi
echo "KEY URI: ${key_uri:0:80}..." | tee -a "$log_file"
echo "KEY IV: $key_iv" | tee -a "$log_file"

# 4. 提取所有分片 URL（以 http 开头的行）
segment_urls=$(echo "$m3u8_content" | grep '^https\?://')
seg_total=$(echo "$segment_urls" | wc -l)
echo "共发现 $seg_total 个分片" | tee -a "$log_file"

if [ "$seg_total" -eq 0 ]; then
    echo "[✗] 没有找到分片 URL" | tee -a "$log_file"
    exit 1
fi

# 5. 逐个下载分片
echo "===== Step 2: 逐个下载并解密分片 =====" | tee -a "$log_file"
i=0
success_count=0
while IFS= read -r seg_url; do
    [ -z "$seg_url" ] && continue
    seg_idx=$(printf "%03d" $i)
    seg_m3u8="${tmp_dir}/${raw_file_name}_${seg_idx}.m3u8"
    seg_mp4="${tmp_dir}/${raw_file_name}_seg${seg_idx}.mp4"

    # 构造单分片 m3u8
    cat > "$seg_m3u8" <<EOF
#EXTM3U
#EXT-X-VERSION:7
#EXT-X-ALLOW-CACHE:NO
#EXT-X-TARGETDURATION:35
#EXT-X-KEY:METHOD=AES-128,URI="${key_uri}",IV=${key_iv}
#EXTINF:3.0,
${seg_url}
#EXT-X-ENDLIST
EOF

    echo "  下载分片 $i ..." | tee -a "$log_file"
    ffmpeg -allowed_extensions ALL \
           -allowed_segment_extensions ALL \
           -extension_picky 0 \
           -protocol_whitelist "file,http,https,tcp,tls,crypto" \
           -i "$seg_m3u8" \
           -c copy \
           -y \
           "$seg_mp4" \
           >>"$log_file" 2>&1

    if [ $? -eq 0 ] && [ -f "$seg_mp4" ]; then
        echo "  [✓] 分片 $i 下载成功: $(du -h "$seg_mp4" | cut -f1)" | tee -a "$log_file"
        echo "file '$seg_mp4'" >> "$seg_list"
        success_count=$((success_count + 1))
    else
        echo "  [✗] 分片 $i 下载失败" | tee -a "$log_file"
    fi

    rm -f "$seg_m3u8"
    i=$((i + 1))
done <<< "$segment_urls"

echo "成功下载 $success_count / $seg_total 个分片" | tee -a "$log_file"

if [ "$success_count" -eq 0 ]; then
    echo "[✗] 没有成功下载任何分片" | tee -a "$log_file"
    rm -f "$seg_list"
    exit 1
fi

# 6. 用 concat demuxer 拼接所有分片
echo "===== Step 3: 拼接分片为最终 MP4 =====" | tee -a "$log_file"
ffmpeg -f concat \
       -safe 0 \
       -i "$seg_list" \
       -c copy \
       -movflags +faststart \
       -y \
       "$file_name" \
       2>&1 | tee -a "$log_file"

concat_exit=${PIPESTATUS[0]}

# 7. 清理临时文件
rm -f "${tmp_dir}/${raw_file_name}"_seg*.mp4 "$seg_list"

# 8. 检查是否成功
if [ $concat_exit -eq 0 ]; then
    echo "[✓] 下载成功: $file_name" | tee -a "$log_file"
    echo "[i] 详细日志: $log_file" | tee -a "$log_file"
else
    echo "[✗] 拼接失败 (ffmpeg 退出码: $concat_exit)，删除残留文件" | tee -a "$log_file"
    echo "[i] 详细日志: $log_file" | tee -a "$log_file"
    rm -f "$file_name"
fi

```

### 第二步

在`configuration.yaml`最新吗增加shell命令

把`camera.midr_ph300_2112_video_doorbell`替换成你的实体ID

```shell
shell_command:
  xiaomi_autosave: '/bin/bash /config/xiaomi_video_autosave.sh "{{state_attr("camera.midr_ph300_2112_video_doorbell","stream_address")}}" "{{state_attr("camera.midr_ph300_2112_video_doorbell","motion_video_time")}}"'
```

### 第三步

增加自动化脚本，在`automations.yaml`最下面放入：

```shell
- id: '1678104080023'
  alias: door_video_autosave
  description: ''
  trigger:
  - platform: state
    entity_id:
    - camera.midr_ph300_2112_video_doorbell ## 将此处修改为你的camera实体
    attribute: motion_video_time
  condition: []
  action:
  - service: shell_command.xiaomi_autosave
    data: {}
```

以上！
