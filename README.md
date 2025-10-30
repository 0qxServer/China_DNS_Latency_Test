# 中国DNS以及常见DNS延迟测试
# 将以下内容复制到powershell 执行，自动输出结果。

# DNS 解析延迟测试

Clear-Host
Write-Host "===============================================" -ForegroundColor Cyan
Write-Host "        中国与全球常用 DNS 解析速度测试        " -ForegroundColor Cyan
Write-Host "===============================================" -ForegroundColor Cyan
Write-Host ""

# 测试域名（选一些常见网站）
$domains = @("www.baidu.com", "www.bilibili.com", "www.google.com", "www.cloudflare.com", "www.youtube.com")

# 公网 DNS 列表
$dnsList = @{
    "114DNS(114.114.114.114)" = "114.114.114.114"
    "阿里DNS(223.5.5.5)" = "223.5.5.5"
    "阿里DNS备用(223.6.6.6)" = "223.6.6.6"
    "腾讯DNS(119.29.29.29)" = "119.29.29.29"
    "百度DNS(180.76.76.76)" = "180.76.76.76"
    "CNNIC DNS(1.2.4.8)" = "1.2.4.8"
    "Google DNS(8.8.8.8)" = "8.8.8.8"
    "Google DNS2(8.8.4.4)" = "8.8.4.4"
    "Cloudflare(1.1.1.1)" = "1.1.1.1"
    "Quad9(9.9.9.9)" = "9.9.9.9"
    "OpenDNS(208.67.222.222)" = "208.67.222.222"
}

$iterations = 3   # 每个域名解析次数
$results = @()

# 开始测试
$totalTests = $domains.Length * $dnsList.Count * $iterations
$progress = 0

foreach ($dns in $dnsList.GetEnumerator()) {
    $serverName = $dns.Key
    $serverIP = $dns.Value
    Write-Host "`n正在测试: $serverName ($serverIP)" -ForegroundColor Yellow

    $totalTime = 0
    $successCount = 0
    $testCount = 0

    foreach ($domain in $domains) {
        for ($i=1; $i -le $iterations; $i++) {
            $testCount++
            $start = Get-Date
            try {
                $null = Resolve-DnsName -Server $serverIP -Name $domain -ErrorAction Stop
                $end = Get-Date
                $diff = ($end - $start).TotalMilliseconds
                $totalTime += $diff
                $successCount++
            } catch {
                # 解析失败
            }

            # 更新进度条
            $progress++
            $percent = [math]::Round(($progress / $totalTests) * 100, 2)
            Write-Progress -PercentComplete $percent -Status "$serverName 测试中" -Activity "正在测试 $domain 第 $i 次"
        }
    }

    if ($successCount -gt 0) {
        $avg = [math]::Round($totalTime / $successCount, 2)
        Write-Host (" → 平均耗时: {0} ms (成功 {1}/{2})" -f $avg, $successCount, $testCount) -ForegroundColor Green
        $results += [PSCustomObject]@{
            DNS = $serverName
            IP = $serverIP
            AvgMs = $avg
            Success = "$successCount/$testCount"
        }
    } else {
        Write-Host " → 全部解析失败！" -ForegroundColor Red
        $results += [PSCustomObject]@{
            DNS = $serverName
            IP = $serverIP
            AvgMs = 9999
            Success = "0/$testCount"
        }
    }
}

# 排序并输出总结
Write-Host "`n===============================================" -ForegroundColor Cyan
Write-Host "               排序结果（由快到慢）" -ForegroundColor Cyan
Write-Host "===============================================" -ForegroundColor Cyan

$results | Sort-Object AvgMs | Format-Table -AutoSize

# 可选：保存到文件
$outfile = "dns_test_result.txt"
$timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
"测试时间：$timestamp" | Out-File $outfile -Encoding utf8
$results | Sort-Object AvgMs | Out-File -Append $outfile -Encoding utf8

Write-Host "`n结果已保存到 $outfile" -ForegroundColor Cyan
Pause
