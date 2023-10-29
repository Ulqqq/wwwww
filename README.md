# Streamlink GUI 2.04

Add-Type -Assembly System.Windows.Forms
[System.Windows.Forms.Application]::EnableVisualStyles()


#### Load Settings and Paths Testing ##############################################################################################################################################

$settings_txt = "settings.txt"

if (Test-Path $settings_txt) {
  Get-Content $settings_txt | ForEach-Object -Begin {$settings=@{}} -Process {$i = $_.Split("="); if(($i[0].CompareTo("") -ne 0) -and ($i[0].StartsWith("[") -ne $true) -and ($i[0].StartsWith("#") -ne $true)) {$settings.Add($i[0], $i[1])}}
} else {
  $msgBoxInput = [System.Windows.Forms.MessageBox]::Show("$settings_txt not found", "Error", "OK", "Exclamation"); if($msgBoxInput -eq "OK"){Break}
}

$hide_powershell_console = $settings.HidePowerShellConsole

if ($hide_powershell_console -eq 1) {
Add-Type -Name Window -Namespace Console -MemberDefinition '
[DllImport("Kernel32.dll")]
public static extern IntPtr GetConsoleWindow();
[DllImport("user32.dll")]
public static extern bool ShowWindow(IntPtr hWnd, Int32 nCmdShow);
'
$consolePtr = [Console.Window]::GetConsoleWindow()
[Console.Window]::ShowWindow($consolePtr, 0)
}

$enable_recording = $settings.EnableRecording
$streamlink_exe = $settings.StreamlinkPath

if ($enable_recording -eq 1 -and [bool] (Get-Command -ErrorAction Ignore -Type Application $streamlink_exe) -eq $false -and (Test-Path $streamlink_exe) -eq $false) {
  $msgBoxInput = [System.Windows.Forms.MessageBox]::Show("$streamlink_exe not found", "Error", "OK", "Exclamation"); if($msgBoxInput -eq "OK"){Break}
}

$curl_exe = $settings.CurlPath

if ([bool] (Get-Command -ErrorAction Ignore -Type Application $curl_exe) -eq $false -and (Test-Path $curl_exe) -eq $false) {
  $msgBoxInput = [System.Windows.Forms.MessageBox]::Show("$curl_exe not found", "Error", "OK", "Exclamation"); if($msgBoxInput -eq "OK"){Break}
}

$load_model_list_on_startup = $settings.LoadModelListOnStartup
$model_list_path = $settings.ModelListPath

if ($load_model_list_on_startup -eq 1 -and -not (Test-Path $model_list_path)) {
  [System.Windows.Forms.MessageBox]::Show("$model_list_path not found", "Information", "OK", "Information")
} else {
  if (Test-Path $model_list_path) {
    Copy-Item -Path "$model_list_path" -Destination "$model_list_path.bac"
  }
}

$to_use_proxy_list = $settings.UseProxyListForGeoBlockModels
$proxy_list_path = $settings.ProxyListPath

if ($to_use_proxy_list -eq 1 -and -not (Test-Path $proxy_list_path)) {
  $msgBoxInput = [System.Windows.Forms.MessageBox]::Show("$proxy_list_path not found", "Error", "OK", "Exclamation"); if($msgBoxInput -eq "OK"){Break}
}

$save_directory = $settings.SaveDirectory

if ($enable_recording -eq 1 -and -not (Test-Path $save_directory)) {
  $msgBoxInput = [System.Windows.Forms.MessageBox]::Show("Folder $save_directory not found. Do you want to create it?", "Question", "YesNo", "Question")
  if($msgBoxInput -eq "Yes") {New-Item -Path $save_directory -ItemType Directory} else {Break}
}

$ps_major_version = $PSVersionTable.PSVersion.Major
$ps_minor_version = $PSVersionTable.PSVersion.Minor
if ($ps_major_version -and $ps_major_version -lt 5) {
  [System.Windows.Forms.MessageBox]::Show("Recommended to upgrade PowerShell. Current version: $ps_major_version.$ps_minor_version", "Information", "OK", "Information")
}

$user_agent = $settings.UserAgent
$waiting_time = $settings.WaitingTimeBetweenAttemptsSeconds
$start_rec_delay = $settings.DelayBetweenStartRecordingsMilliSeconds
$start_rec_after_add = $settings.StartRecAfterAddingToList
$separate_directory = $settings.SeparateDirectoryForEachModel
$url_chaturbate = $settings.UrlChaturbate
$url_bongacams = $settings.UrlBongacams
$url_stripchat = $settings.UrlStripchat
$url_myfreecams = $settings.UrlMyFreeCams
$show_notify_icon = $settings.ShowNotifyIcon

$cookie_bonga20120608 = $settings.CookieBonga20120608
$x_ab_split_group = $settings.XAbSplitGroup

$always_on_top = $settings.MainFormAlwaysOnTop
$center_screen_position = $settings.MainFormStartPositionCenterScreen
$location_x = $settings.MainFormStartPositionX
$location_y = $settings.MainFormStartPositionY

$check_status_after_start = $settings.CheckModelStatusAfterRecStart
$status_check_delay = $settings.DelayBetweenStatusChecksMilliSeconds
$auto_status_check_when_minimized = $settings.AutoStatusCheckWhenMinimized
$status_check_interval = $settings.AutoStatusCheckIntervalMinutes
$enable_notifications = $settings.EnableNotifications
$start_rec_if_online = $settings.StartRecordingIfStatusOnline

$start_rec_after_load_list = $settings.StartRecAfterLoadList
$autosave_model_list = $settings.AutoSaveModelList

$to_use_proxy_list_for_mfc_status_check = $settings.UseProxyListForMyFreeCamsStatusCheck
$connect_timeout = $settings.ConnectTimeout
$proxy_protocol = $settings.ProxyProtocol

$open_stream_in_vlc = $settings.OpenStreamInVlcMediaPlayer
$vlc_media_player_path = $settings.VlcMediaPlayerPath
if (-not (Test-Path $vlc_media_player_path)) {
  $vlc_not_found = 1
}
$vlc_network_caching = $settings.VlcNetworkCachingMilliSeconds

$open_records_folder_in_totalcmd = $settings.OpenSaveDirectoryInTotalCommander
$totalcmd_path = $settings.TotalCommanderPath

$cb_thum_animation_speed = $settings.ChaturbateThumbnailAnimationSpeedMilliseconds
$bc_thum_animation_speed = $settings.BongaCamsThumbnailAnimationSpeedMilliseconds

#### Test Connection ##############################################################################################################################################################

function TestConnection {
  $command = {
    param ($curl_exe, $user_agent, $url_chaturbate, $url_bongacams, $url_stripchat, $url_myfreecams, $cookie_bonga20120608, $x_ab_split_group)

    $model_name = 'model'

    $chaturbate_content_type = & $curl_exe --silent --show-error --insecure --user-agent $user_agent --write-out '%{content_type}' --output 'nul' --url "$url_chaturbate/api/chatvideocontext/$model_name/"
    if ($chaturbate_content_type -eq 'application/json') {
      Write-Host "UrlChaturbate=$url_chaturbate - OK"
    } else {
      Write-Host "UrlChaturbate=$url_chaturbate - ERROR" -ForegroundColor red -BackgroundColor black
    }
    Write-Host "`n"

    Write-Host "------------------------------------------------------"
    $bongacams_content_type = & $curl_exe --silent --show-error --insecure --user-agent $user_agent --write-out '%{content_type}' --output 'nul' --header 'X-ab-Split-Group: $x_ab_split_group' --header 'Cookie: bonga20120608=$cookie_bonga20120608' --header 'X-Requested-With: XMLHttpRequest' --data "method=getRoomData'&'args%5B%5D=$model_name" --url "$url_bongacams/tools/amf.php"
    if ($bongacams_content_type -eq 'application/json') {
      Write-Host "UrlBongacams=$url_bongacams - OK"
    } else {
      Write-Host "UrlBongacams=$url_bongacams - ERROR" -ForegroundColor red -BackgroundColor black
    }
    Write-Host "`n"

    Write-Host "------------------------------------------------------"
    $stripchat_content_type = & $curl_exe --silent --show-error --insecure --user-agent $user_agent --write-out '%{content_type}' --output 'nul' --url "$url_stripchat/api/front/models/username/$model_name/cam/"
    if ($stripchat_content_type -eq 'application/json') {
      Write-Host "UrlStripchat=$url_stripchat - OK"
    } else {
      Write-Host "UrlStripchat=$url_stripchat - ERROR" -ForegroundColor red -BackgroundColor black
    }
    Write-Host "`n"

    Write-Host "------------------------------------------------------"
    $myfreecams_content_type = & $curl_exe --silent --location --show-error --insecure --user-agent $user_agent --write-out '%{content_type}' --output 'nul' --url "$url_myfreecams"
    if ($myfreecams_content_type -eq 'text/html; charset=UTF-8') {
      Write-Host "UrlMyFreeCams=$url_myfreecams - OK"
    } else {
      Write-Host "UrlMyFreeCams=$url_myfreecams - ERROR" -ForegroundColor red -BackgroundColor black
    }
    Write-Host "`n"

  }
  Start-Process PowerShell -ArgumentList "-NoExit -Command (Invoke-Command -ScriptBlock {$command} -ArgumentList '$curl_exe', '$user_agent', '$url_chaturbate', '$url_bongacams', '$url_stripchat', '$url_myfreecams', '$cookie_bonga20120608', '$x_ab_split_group')"
}

###################################################################################################################################################################################
#### Start Recording ##############################################################################################################################################################
###################################################################################################################################################################################

function StartRecording ($job_name, $site, $model_name, $stream_quality) {

$recordChaturbate = {

  param ($url_chaturbate, $model_name, $stream_quality, $save_directory, $curl_exe, $user_agent, $streamlink_exe, $waiting_time, $separate_directory, $proxy_list_path, $connect_timeout, $proxy_protocol, $to_use_proxy_list)

  while($true) {
    $curlArguments1 = "--silent", "--show-error", "--insecure", 
                      "--user-agent", "$user_agent", 
                      "--url", "$url_chaturbate/api/chatvideocontext/$model_name/"
    $json = $null
    $json = & $curl_exe @curlArguments1 | ConvertFrom-Json
    $room_status = $null
    $room_status = $json.room_status
    if ($room_status -eq "public") {
      $stream_url = $json.hls_source
      $date_time = Get-Date -format MM-dd_HH-mm-ss
      if ($stream_quality -eq "best") {
        $quality = $null
      } elseif ($stream_quality -match ",") {
        $quality = "-other"
      } else {
        $quality = "-" + $stream_quality
      }
      if ($separate_directory -eq 1 -and $save_directory -notmatch $model_name) {
        $save_directory = $save_directory + $model_name + "\"
        if (!(Test-Path $save_directory)) {New-Item -Path $save_directory -ItemType Directory}
      }
      $output_file = $save_directory + "0Chaturbate-" + $model_name + "-" + $date_time + $quality + ".ts" ####### $date_time "Chaturbate-"
      & $streamlink_exe --stream-segment-threads 3 --hls-duration 01:00:00 --http-timeout 10 --hls-timeout 20 --default-stream $stream_quality --url $stream_url --output $output_file
    }

    if ($to_use_proxy_list -eq 1 -and ($json.detail -match "region" -or $json -eq $null)) {
      $curlArguments2 = "--silent", "--show-error", "--insecure", 
                        "--user-agent", "$user_agent", 
                        "--write-out", "%{http_code}", 
                        "--output", "nul", 
                        "--url", "https://cbjpeg.stream.highwebmedia.com/stream?room=$model_name"
      $status_code = & $curl_exe @curlArguments2
      if ($status_code -eq "200") {
        if (Test-Path $proxy_list_path) {
          $proxy_list = @(Get-Content -Path $proxy_list_path)
        }
        ForEach ($proxy in $proxy_list) {
          switch ($proxy_protocol) {
            "socks4" {$proxy_with_prefix = "socks4://" + $proxy}
            "socks4a" {$proxy_with_prefix = "socks4a://" + $proxy}
            "socks5" {$proxy_with_prefix = "socks5://" + $proxy}
            "socks5h" {$proxy_with_prefix = "socks5h://" + $proxy}
            Default {$proxy_with_prefix = $proxy}
          }
          $curlArguments3 = "--silent", "--insecure", 
                            "--user-agent", "$user_agent", 
                            "--proxy", "$proxy_with_prefix", 
                            "--connect-timeout", "$connect_timeout", 
                            "--url", "$url_chaturbate/api/chatvideocontext/$model_name/"
          $json = & $curl_exe @curlArguments3 | ConvertFrom-Json
          if ($json.room_status -eq "public") {
            $stream_url = $json.hls_source
            $date_time = Get-Date -format MM-dd_HH-mm-ss
            if ($stream_quality -eq "best") {
              $quality = $null
            } elseif ($stream_quality -match ",") {
              $quality = "-other"
            } else {
              $quality = "-" + $stream_quality
            }
            if ($separate_directory -eq 1 -and $save_directory -notmatch $model_name) {
              $save_directory = $save_directory + $model_name + "\"
              if (!(Test-Path $save_directory)) {New-Item -Path $save_directory -ItemType Directory}
            }
            $output_file = $save_directory + "0Chaturbate-" + $model_name + "-" + $date_time + $quality + "-proxy.ts" 
            & $streamlink_exe --stream-segment-threads 3 --hls-duration 01:00:00 --http-timeout 10 --hls-timeout 20 --default-stream $stream_quality --url $stream_url --output $output_file
          }
        }
      } 
    }
  Start-Sleep -seconds $waiting_time
  }
}

$recordBongaCams = {

  param ($url_bongacams, $model_name, $stream_quality, $save_directory, $curl_exe, $user_agent, $streamlink_exe, $waiting_time, $separate_directory, $proxy_list_path, $connect_timeout, $proxy_protocol, $to_use_proxy_list, $cookie_bonga20120608, $x_ab_split_group)

  while($true) {
    $curlArguments1 = "--silent", "--show-error", "--location", "--head", 
                      "--user-agent", "$user_agent", 
                      "--write-out", "%{url_effective}", 
                      "--output", "nul", 
                      "--url", "$url_bongacams/$model_name"
    $redirectedUrl = $null
    $redirectedUrl = & $curl_exe @curlArguments1
    if ($redirectedUrl -notmatch "/profile/") {
      $curlArguments2 = "--silent", "--show-error", "--insecure", 
                        "--user-agent", "$user_agent", 
                        "--header", "X-ab-Split-Group: $x_ab_split_group", 
                        "--header", "Cookie: bonga20120608=$cookie_bonga20120608", 
                        "--header", "X-Requested-With: XMLHttpRequest", 
                        "--data", "method=getRoomData&args%5B%5D=$model_name", 
                        "--url", "$url_bongacams/tools/amf.php"
      $json = $null
      $json = & $curl_exe @curlArguments2 | ConvertFrom-Json
      if ($json.performerData.showType -eq "public") {
        $view_server = $json.localData.videoServerUrl
        $username = $json.performerData.username
        $stream_url = "https:" + $view_server + "/hls/stream_" + $username + "/playlist.m3u8"
        $date_time = Get-Date -format MM-dd_HH-mm-ss
        if ($stream_quality -eq "best" -or $stream_quality -match ",") {
          $quality = $null
        } else {
          $quality = "-" + $stream_quality
        }
        if ($separate_directory -eq 1 -and $save_directory -notmatch $model_name) {
          $save_directory = $save_directory + $model_name + "\"
          if (!(Test-Path $save_directory)) {New-Item -Path $save_directory -ItemType Directory}
        }
        $output_file = $save_directory + "0BongaCams-" + $model_name + "-" + $date_time + $quality + ".ts"
        & $streamlink_exe --stream-segment-threads 3 --hls-duration 01:00:00 --http-timeout 10 --hls-timeout 20 --default-stream $stream_quality --url $stream_url --output $output_file
      }
    }

    if ($to_use_proxy_list -eq 1 -and ($json.status -eq "error" -or $redirectedUrl -eq $null)) {
      if (Test-Path $proxy_list_path) {
        $proxy_list = @(Get-Content -Path $proxy_list_path)
      }
      ForEach ($proxy in $proxy_list) {
        switch ($proxy_protocol) {
          "socks4" {$proxy_with_prefix = "socks4://" + $proxy}
          "socks4a" {$proxy_with_prefix = "socks4a://" + $proxy}
          "socks5" {$proxy_with_prefix = "socks5://" + $proxy}
          "socks5h" {$proxy_with_prefix = "socks5h://" + $proxy}
          Default {$proxy_with_prefix = $proxy}
        }
        $curlArguments3 = "--silent", "--insecure", 
                          "--user-agent", "$user_agent", 
                          "--header", "X-ab-Split-Group: $x_ab_split_group", 
                          "--header", "Cookie: bonga20120608=$cookie_bonga20120608", 
                          "--header", "X-Requested-With: XMLHttpRequest", 
                          "--data", "method=getRoomData&args%5B%5D=$model_name", 
                          "--proxy", "$proxy_with_prefix", 
                          "--connect-timeout", "$connect_timeout", 
                          "--url", "$url_bongacams/tools/amf.php"
        $json = & $curl_exe @curlArguments3 | ConvertFrom-Json
        if ($json.performerData.showType -eq "public") {
          $view_server = $json.localData.videoServerUrl
          $stream_url = "https:" + $view_server + "/hls/stream_" + $model_name + "/playlist.m3u8"
          $date_time = Get-Date -format MM-dd_HH-mm-ss
          if ($stream_quality -eq "best" -or $stream_quality -match ",") {
            $quality = $null
          } else {
            $quality = "-" + $stream_quality
          }
          if ($separate_directory -eq 1 -and $save_directory -notmatch $model_name) {
            $save_directory = $save_directory + $model_name + "\"
            if (!(Test-Path $save_directory)) {New-Item -Path $save_directory -ItemType Directory}
          }
          $output_file = $save_directory + "0BongaCams-" + $model_name + "-" + $date_time + $quality + "-proxy.ts"
          & $streamlink_exe --stream-segment-threads 3 --hls-duration 01:00:00 --http-timeout 10 --hls-timeout 20 --default-stream $stream_quality --url $stream_url --output $output_file
        }
      }
    }
  Start-Sleep -seconds $waiting_time
  }
}

$recordStripChat = {

  param ($url_stripchat, $model_name, $stream_quality, $save_directory, $curl_exe, $user_agent, $streamlink_exe, $waiting_time, $separate_directory, $proxy_list_path, $connect_timeout, $proxy_protocol, $to_use_proxy_list)

  switch ($stream_quality) {
    "best" {$quality = $null}
    "720p" {$quality = "_720p"}
    "480p" {$quality = "_480p"}
    "240p" {$quality = "_240p"}
    Default {$quality = $null}
  }

  while($true) {
    $curlArguments1 = "--silent", "--show-error", "--insecure", 
                      "--user-agent", "$user_agent", 
                      "--url", "$url_stripchat/api/front/models/username/$model_name/cam/"
    $json_cam = $null
    $json_cam = & $curl_exe @curlArguments1 | ConvertFrom-Json

#### proxy ################################

    if ($to_use_proxy_list -eq 1 -and $json_cam -eq $null) {
      if (Test-Path $proxy_list_path) {
        $proxy_list = @(Get-Content -Path $proxy_list_path)
      }
      ForEach ($proxy in $proxy_list) {
        switch ($proxy_protocol) {
          "socks4" {$proxy_with_prefix = "socks4://" + $proxy}
          "socks4a" {$proxy_with_prefix = "socks4a://" + $proxy}
          "socks5" {$proxy_with_prefix = "socks5://" + $proxy}
          "socks5h" {$proxy_with_prefix = "socks5h://" + $proxy}
          Default {$proxy_with_prefix = $proxy}
        }
        $curlArguments3 = "--silent", "--show-error", "--insecure", 
                          "--user-agent", "$user_agent", 
                          "--proxy", "$proxy_with_prefix", 
                          "--connect-timeout", "$connect_timeout", 
                          "--url", "$url_stripchat/api/front/models/username/$model_name/cam/"
        $json_cam = $null
        $json_cam = & $curl_exe @curlArguments3 | ConvertFrom-Json
        $isCamAvailable = $null
        $isCamAvailable = $json_cam.isCamAvailable
        if ($isCamAvailable -match "True") {
          $view_server = $json_cam.viewServers."flashphoner-hls"
          $curlArguments4 = "--silent", "--insecure", 
                            "--user-agent", "$user_agent", 
                            "--proxy", "$proxy_with_prefix", 
                            "--connect-timeout", "$connect_timeout", 
                            "--url", "$url_stripchat/api/front/users/username/$model_name/"
          $model_id = (& $curl_exe @curlArguments4 | ConvertFrom-Json).user.id
          $date_time = get-date -format MM-dd_HH-mm-ss
          if ($separate_directory -eq 1 -and $save_directory -notmatch $model_name) {
            $save_directory = $save_directory + $model_name + "\"
            if (!(Test-Path $save_directory)) {New-Item -Path $save_directory -ItemType Directory}
          }
          $output_file = $save_directory + "StripChat-" + $model_name + "-" + $date_time + $quality + "-proxy.ts"
          $stream_url = "https://b-" + $view_server + ".stripst.com/hls/" + $model_id + $quality + "/" + $model_id + $quality + ".m3u8"
          & $streamlink_exe --stream-segment-threads 3 --hls-duration 01:00:00 --http-timeout 10 --hls-timeout 20 --default-stream best --url $stream_url --output $output_file
        }
      }
    }

###########################################

    $isCamAvailable = $null
    $isCamAvailable = $json_cam.isCamAvailable
    if ($isCamAvailable -match "True") {
      $view_server = $json_cam.viewServers."flashphoner-hls"
      $curlArguments2 = "--silent", "--show-error", "--insecure", 
                        "--user-agent", "$user_agent", 
                        "--url", "$url_stripchat/api/front/users/username/$model_name/"
      $model_id = (& $curl_exe @curlArguments2 | ConvertFrom-Json).user.id
      $date_time = get-date -format MM-dd_HH-mm-ss
      if ($separate_directory -eq 1 -and $save_directory -notmatch $model_name) {
        $save_directory = $save_directory + $model_name + "\"
        if (!(Test-Path $save_directory)) {New-Item -Path $save_directory -ItemType Directory}
      }
      $output_file = $save_directory + "StripChat-" + $model_name + "-" + $date_time + $quality + ".ts"
      $stream_url = "https://b-" + $view_server + ".stripst.com/hls/" + $model_id + $quality + "/" + $model_id + $quality + ".m3u8"
      & $streamlink_exe --stream-segment-threads 3 --hls-duration 01:00:00 --http-timeout 10 --hls-timeout 20 --default-stream best --url $stream_url --output $output_file
    }
  Start-Sleep -seconds $waiting_time
  }
}

if ($site -eq "MyFreeCams") {
  $url_myfreecams_profiles = $url_myfreecams -replace "//", "//profiles."
  $curlArguments9 = "--silent", "--show-error", "--insecure", "--user-agent", "$user_agent", "--url", "$url_myfreecams_profiles/$model_name"
  $profile_page = $null
  $profile_page = & $curl_exe @curlArguments9
  if ($($profile_page | Select-String -Pattern "profile is currently unavailable")) {
    [System.Windows.Forms.MessageBox]::Show("Not supported for geo-block models", "$model_name - $site", "OK", "Information")
  }
}

$recordMyFreeCams = {

  param ($url_myfreecams, $model_name, $stream_quality, $save_directory, $curl_exe, $user_agent, $streamlink_exe, $waiting_time, $separate_directory, $proxy_list_path, $connect_timeout, $proxy_protocol, $to_use_proxy_list)

  while($true) {
    $url_myfreecams_profiles = $url_myfreecams -replace "//", "//profiles."
    $curlArguments1 = "--silent", "--show-error", "--insecure", 
                      "--user-agent", "$user_agent", 
                      "--url", "$url_myfreecams_profiles/$model_name"
    $profile_page = $null
    $profile_page = & $curl_exe @curlArguments1
    $profile_state_string = $null
    $profile_state_string = $profile_page | Select-String -Pattern 'profileState: {"number"'
    $status = $null
    if ($profile_state_string -match "null") {
      $status = "null"
    } elseif ($profile_state_string) {
      $status = $($profile_state_string.Line.Split('"')[5]).ToLower()
    } elseif ($($profile_page | Select-String -Pattern "profile is currently unavailable")) {
      $status = "geo ban"
    } else {
      $status = "null"
    }
    $status = $status -replace "online - webcam off", "camoff"

    if ($status -eq "online") {
      $date_time = get-date -format MM-dd_HH-mm-ss
      if ($separate_directory -eq 1 -and $save_directory -notmatch $model_name) {
        $save_directory = $save_directory + $model_name + "\"
        if (!(Test-Path $save_directory)) {New-Item -Path $save_directory -ItemType Directory}
      }
      $output_file = $save_directory + "MyFreeCams-" + $model_name + "-" + $date_time + ".ts"
      $url = $url_myfreecams + "/#" + $model_name
      & $streamlink_exe --stream-segment-threads 3 --hls-duration 01:00:00 --http-timeout 10 --hls-timeout 20 --default-stream best --url $url --output $output_file
    }
    Start-Sleep -seconds $waiting_time
  }
}

  $job_arguments_cb= @($url_chaturbate, $model_name, $stream_quality, $save_directory, $curl_exe, $user_agent, $streamlink_exe, $waiting_time, $separate_directory, $proxy_list_path, $connect_timeout, $proxy_protocol, $to_use_proxy_list)
  $job_arguments_bc= @($url_bongacams, $model_name, $stream_quality, $save_directory, $curl_exe, $user_agent, $streamlink_exe, $waiting_time, $separate_directory, $proxy_list_path, $connect_timeout, $proxy_protocol, $to_use_proxy_list, $cookie_bonga20120608, $x_ab_split_group)
  $job_arguments_sc= @($url_stripchat, $model_name, $stream_quality, $save_directory, $curl_exe, $user_agent, $streamlink_exe, $waiting_time, $separate_directory, $proxy_list_path, $connect_timeout, $proxy_protocol, $to_use_proxy_list)
  $job_arguments_mfc= @($url_myfreecams, $model_name, $stream_quality, $save_directory, $curl_exe, $user_agent, $streamlink_exe, $waiting_time, $separate_directory, $proxy_list_path, $connect_timeout, $proxy_protocol, $to_use_proxy_list)

  switch ($site) {
   "Chaturbate" {Start-Job -Init ([ScriptBlock]::Create("Set-Location '$pwd'")) -Name $job_name -ScriptBlock $recordChaturbate -ArgumentList $job_arguments_cb}
   "BongaCams" {Start-Job -Init ([ScriptBlock]::Create("Set-Location '$pwd'")) -Name $job_name -ScriptBlock $recordBongaCams -ArgumentList $job_arguments_bc}
   "StripChat" {Start-Job -Init ([ScriptBlock]::Create("Set-Location '$pwd'")) -Name $job_name -ScriptBlock $recordStripChat -ArgumentList $job_arguments_sc}
   "MyFreeCams" {Start-Job -Init ([ScriptBlock]::Create("Set-Location '$pwd'")) -Name $job_name -ScriptBlock $recordMyFreeCams -ArgumentList $job_arguments_mfc}
  }

}

###################################################################################################################################################################################
#### Get Model Status #############################################################################################################################################################
###################################################################################################################################################################################

function GetModelStatus ($site, $model_name) {
  if ($site -eq "Chaturbate") {
    $curlArguments1 = "--silent", "--show-error", "--insecure", 
                      "--user-agent", "$user_agent", 
                      "--connect-timeout", "$connect_timeout", 
                      "--url", "$url_chaturbate/api/chatvideocontext/$model_name/"
    $curlArguments2 = "--silent", "--show-error", "--insecure", 
                      "--user-agent", "$user_agent", 
                      "--write-out", "%{http_code}", 
                      "--connect-timeout", "$connect_timeout", 
                      "--output", "nul", 
                      "--url", "https://cbjpeg.stream.highwebmedia.com/stream?room=$model_name"
    $json= $null
    $json = & $curl_exe @curlArguments1 | ConvertFrom-Json
    if ($json -and $json.status -ne "401") {
      $status = $json.room_status
    } elseif ($json.status -eq "401" -and $json.detail -match "banned") {
      $status = "banned"
    } elseif ($json.status -eq "401" -and $json.detail -match "password") {
      $status = "password"
    } elseif ($json.status -eq "401" -and $json.detail -match "deleted") {
      $status = "deleted"
    } else {
      $status_code = $null
      $status_code = & $curl_exe @curlArguments2
      if ($json.status -eq "401" -and $json.detail -match "region") {
        $geo_ban = ", geo ban"
      } elseif (!$json) {
        $geo_ban = ", null"
      } else {
        $geo_ban = $null
      }
      switch  ($status_code) {
        "200" {$status = "public" + $geo_ban}
        "204" {$status = "offline" + $geo_ban}
        "403" {$status = "private" + $geo_ban}
        "000" {$status = "null" + $geo_ban}
        Default {$status = $status_code}
      }
    }
  }

  if ($site -eq "BongaCams") {
  $curl_arg_bc_chat_data = "--silent", "--show-error", "--insecure", 
                           "--user-agent", "$user_agent", 
                           "--header", "X-Requested-With: XMLHttpRequest", 
                           "--url", "$url_bongacams/get-member-chat-data?username=$model_name"
  $response = $null
  $response = & $curl_exe @curl_arg_bc_chat_data
  if ($response) {
    $json = $null
    $json = $response | ConvertFrom-Json
    if ($json) {
      if ($json.status -eq "error") {
        $status = "error"
        $curl_arg_bc_redirected = "--silent", "--show-error", "--location", "--head", "--insecure", 
                                  "--user-agent", "$user_agent", 
                                  "--write-out", "%{url_effective}", 
                                  "--output", "nul", 
                                  "--url", "$url_bongacams/$model_name"
        $redirected_url = $null
        $redirected_url = & $curl_exe @curl_arg_bc_redirected
        if ($redirected_url -match "{0}_del" -f $model_name) {
          $status = "deleted"
        } elseif ($redirected_url -match "/profile/") {
          $status = "offline"
       }
      } elseif ($json.status -eq "success") {
        $is_offline = $json.result.chatShowStatusOptions.isOffline
        $is_privat_chat = $json.result.chatShowStatusOptions.isPrivatChat
        $is_full_privat_chat = $json.result.chatShowStatusOptions.isFullPrivatChat
        $is_group_privat_chat = $json.result.chatShowStatusOptions.isGroupPrivatChat
        $is_vip_show = $json.result.chatShowStatusOptions.isVipShow
        if ($is_offline) {
          $status = "offline"
        } elseif ($is_privat_chat) {
          $status = "private"
        } elseif ($is_full_privat_chat) {
          $status = "fullprivate"
        } elseif ($is_group_privat_chat) {
          $status = "group"
        } elseif ($is_vip_show) {
          $status = "vip"
        } else {
          $status = "public"
        }
      } else {
        $status = $json.status
      }
    } else {
      $status = "null"
    }
  } else {
    $status = "no response"
  }
}

  if ($site -eq "StripChat") {
    $curlArguments5 = "--silent", "--show-error", "--insecure", 
                      "--user-agent", "$user_agent", 
                      "--url", "$url_stripchat/api/front/users/username/$model_name/"
    $curlArguments6 = "--silent", "--show-error", "--insecure", 
                      "--user-agent", "$user_agent", 
                      "--url", "$url_stripchat/api/front/models/username/$model_name/cam/"
    $json = $null
    $json = & $curl_exe @curlArguments5 | ConvertFrom-Json
    $json_cam = $null
    $json_cam = & $curl_exe @curlArguments6 | ConvertFrom-Json
    if (!$json -or !$json_cam) {
      $status = "null"
    } elseif ($json_cam.error -match "Not Found") {
      $status = "not found"
    } elseif ($json.isGeoBanned -match "True") {
      if (-not [string]::IsNullOrEmpty($json_cam.show)) {
        $status = $json_cam.show.mode + ", geo ban"
      } else {
        switch -regex ($json_cam.isCamAvailable) {
          "False" {$status = "off" + ", geo ban"}
          "True" {$status = "public" + ", geo ban"}
        }
      }
    } else {
      $status = $json.user.status
    }
  }

  if ($site -eq "MyFreeCams") {
    $url_myfreecams_profiles = $url_myfreecams -replace "//", "//profiles."
    $curlArguments7 = "--silent", "--show-error", "--insecure", 
                      "--user-agent", "$user_agent", 
                      "--url", "$url_myfreecams_profiles/$model_name"
    $profile_page = $null
    $profile_page = & $curl_exe @curlArguments7
    $profile_state_string = $null
    $profile_state_string = $profile_page | Select-String -Pattern 'profileState: {"number"'
    $status = $null
    $status_proxy = $null
    if ($profile_state_string -match "null") {
      $status = "null"
    } elseif ($profile_state_string) {
      $status = $($profile_state_string.Line.Split('"')[5]).ToLower()
    } elseif ($($profile_page | Select-String -Pattern "profile is currently unavailable")) {
      $status = "geo ban"
    } else {
      $status = "null"
    }
    $status = $status -replace "online - webcam off", "camoff"

    if ($MenuItem_CheckMFCStatusThroughProxy.Checked -eq $true) {
      if (Test-Path $proxy_list_path) {
        $proxy_list = @(Get-Content -Path $proxy_list_path)
      }
    }

    if ($MenuItem_CheckMFCStatusThroughProxy.Checked -eq $true -and ($status -eq "geo ban" -or $status -eq "null")) {
      $first_proxies_number = 3
      if ($proxy_list.Count -lt $first_proxies_number) {
        $first_proxies_number = $proxy_list.Count
      }
      for ($i = 0; $i -lt $first_proxies_number; $i++) {
        switch ($proxy_protocol) {
          "socks4" {$proxy_with_prefix = "socks4://" + $proxy_list[$i]}
          "socks4a" {$proxy_with_prefix = "socks4a://" + $proxy_list[$i]}
          "socks5" {$proxy_with_prefix = "socks5://" + $proxy_list[$i]}
          "socks5h" {$proxy_with_prefix = "socks5h://" + $proxy_list[$i]}
          Default {$proxy_with_prefix = $proxy_list[$i]}
        }
        $curlArguments8 = "--silent", "--show-error", "--insecure", 
                          "--user-agent", "$user_agent", 
                          "--proxy", "$proxy_with_prefix", 
                          "--connect-timeout", "8", 
                          "--url", "$url_myfreecams_profiles/$model_name"

        $profile_page = $null
        $profile_page = & $curl_exe @curlArguments8
        $profile_state_string = $null
        $profile_state_string = $profile_page | Select-String -Pattern 'profileState: {"number"'
        if ($profile_state_string -match "null") {
          $status_proxy = "null"
        } elseif ($profile_state_string) {
          $status_proxy = $($profile_state_string.Line.Split('"')[5]).ToLower()
        } elseif ($($profile_page | Select-String -Pattern "profile is currently unavailable")) {
          $status_proxy = "geo ban"
        } else {
          $status_proxy = "null"
        }
        $status_proxy = $status_proxy -replace "online - webcam off", "camoff"
        if ($status_proxy -ne "null" -and $status_proxy -ne "geo ban") {
          Break
        }
      }
    }
    if ($status_proxy) {
      $status_proxy = $status_proxy + ", "
    }
    $status = $status_proxy + $status
  }

  return $status

}

#### Disabled / Enabled MainForm ##################################################################################################################################################

function MainFormDisabled {
  $MainForm.Icon = $waiting_icon
  if ($show_notify_icon -eq 1) {
    if ($NotifyIcon.Icon -eq $main_icon_green) {
      $NotifyIcon.Icon = $waiting_icon_green
    } else {
      $NotifyIcon.Icon = $waiting_icon
    }
  }
  $MenuStrip.Enabled = $false
}

function MainFormEnabled {
  $MainForm.Icon = $main_icon
  if ($show_notify_icon -eq 1) {
    if ($NotifyIcon.Icon -eq $waiting_icon_green) {
      $NotifyIcon.Icon = $main_icon_green
    } else {
      $NotifyIcon.Icon = $main_icon
    }
  }
  $MenuStrip.Enabled = $true
}

#### Show Status ##################################################################################################################################################################

function DisplayModelStatusInDataGrid ($row) {
  $model_status = GetModelStatus -site $row.Cells[2].Value -model_name $row.Cells[1].Value
  $row.Cells[4].Value = $model_status
  switch -regex ($model_status) {
    "online|public" {$row.DefaultCellStyle.BackColor = "#73FF73"}
    "offline|^off|not found|error|banned|deleted|null" {$row.DefaultCellStyle.BackColor = "#FF7373"}
    Default {$row.DefaultCellStyle.BackColor = "#7373FF"}
  }
}

function CheckStatus {
  if ($DataGridView.SelectedRows.Count -gt 0) {

    $previous_status_hash = @{}
    ForEach ($row in $DataGridView.Rows) {
      if ($row.Cells[4].Value -match "online|public") {
        $previous_status = 1
      } else {
        $previous_status = 0
      }
      $previous_status_hash[$row.Cells[0].Value] = $previous_status
    }
    

    MainFormDisabled
    ForEach ($row in $DataGridView.SelectedRows) {
      DisplayModelStatusInDataGrid -row $row

      if ($row.Cells[4].Value -match "online|public") {
        $current_status = 1
      } else {
        $current_status = 0
      }

      if (($previous_status_hash[$row.Cells[0].Value] -ne $current_status) -and $current_status -ne 0) {
        if ($MenuItem_StartRecordingIfOnline.Checked -eq $true -and [bool] (Get-Job -Name $row.Cells[0].Value -ErrorAction Ignore) -eq $false -and !($row.Cells[2].Value -eq "MyFreeCams" -and $row.Cells[4].Value -match "geo ban")) {
            StartRecording -job_name $row.Cells[0].Value -site $row.Cells[2].Value -model_name $row.Cells[1].Value -stream_quality $row.Cells[3].Value
            $row.Cells[5].Value = $rec_icon
          }
      }
      if ($previous_status_hash[$row.Cells[0].Value] -ne $current_status -and $current_status -ne 1) {
        Stop-Job -Name $row.Cells[0].Value
        Remove-Job -Name $row.Cells[0].Value -Force
        $row.Cells[5].Value = $null
      }

      Start-Sleep -milliseconds $status_check_delay
    }
    MainFormEnabled
  }
  $DataGridView.ClearSelection()
}

function CheckStatusAll {
  if ($DataGridView.RowCount -ne 0) {
    MainFormDisabled


    if ($show_notify_icon -eq 1 -and $MenuItem_CheckStatusWhenMinimized.Checked -eq $true -and $MainForm.WindowState -eq "Minimized") {
      $previous_status_hash = @{}
      ForEach ($row in $DataGridView.Rows) {
        if ($row.Cells[4].Value -match "online|public") {
          $previous_status = 1
        } else {
          $previous_status = 0
        }
        $previous_status_hash[$row.Cells[0].Value] = $previous_status
      }
    }


    $DataGridView.ClearSelection()
    ForEach ($row in $DataGridView.Rows) {
      $row.DefaultCellStyle.BackColor = "#FFFFFF"
      $row.Cells[4].Value = $null
    }
    ForEach ($row in $DataGridView.Rows) {
      DisplayModelStatusInDataGrid -row $row


      if ($row.Cells[4].Value -match "null|000") {
        $curlArguments = "--silent", "--show-error", "--insecure", 
                         "--user-agent", "$user_agent", 
                         "--write-out", "%{http_code}", 
                         "--output", "nul", 
                         "--url", "https://www.google.com/"
        $http_code = & $curl_exe @curlArguments
        if ($http_code -ne 200) {
          if ($show_notify_icon -eq 1) {
            $NotifyIcon.BalloonTipTitle = "Streamlink GUI"
            $NotifyIcon.BalloonTipText = "Check internet connection"
            $NotifyIcon.BalloonTipIcon = "Warning"
            $NotifyIcon.ShowBalloonTip(5000)
          }
          $b = 1
        }
      }
      if ($b -eq 1) {Break}


      if ($MenuItem_CheckStatusWhenMinimized.Checked -eq $true) {
        if ($row.Cells[4].Value -match "online|public") {
          $current_status = 1
        } else {
          $current_status = 0
        }
        if ((!$previous_status_hash -or $previous_status_hash[$row.Cells[0].Value] -ne $current_status) -and $current_status -ne 0) {
          if ($NotifyIcon_MenuItem_EnableNotifications.Checked -eq $true) {
            ShowBalloonTip -site $row.Cells[2].Value -name $row.Cells[1].Value -status $row.Cells[4].Value
          }
          if ($NotifyIcon.Icon -ne $waiting_icon_green) {
            $NotifyIcon.Icon = $waiting_icon_green
          }
          if ($MenuItem_StartRecordingIfOnline.Checked -eq $true -and [bool] (Get-Job -Name $row.Cells[0].Value -ErrorAction Ignore) -eq $false -and !($row.Cells[2].Value -eq "MyFreeCams" -and $row.Cells[4].Value -match "geo ban")) {
            StartRecording -job_name $row.Cells[0].Value -site $row.Cells[2].Value -model_name $row.Cells[1].Value -stream_quality $row.Cells[3].Value
            $row.Cells[5].Value = $rec_icon
          }
        }
        if ($previous_status_hash[$row.Cells[0].Value] -ne $current_status -and $current_status -ne 1) {
          Stop-Job -Name $row.Cells[0].Value
          Remove-Job -Name $row.Cells[0].Value -Force
          $row.Cells[5].Value = $null
        }
      }


      Start-Sleep -milliseconds $status_check_delay
    }

    if ($show_notify_icon -eq 1) {
      $device_id = $save_directory.Split("\")[0]
      $disk_free_space = $(Get-WMIObject Win32_Logicaldisk -filter "DeviceID=`"$device_id`"" | Select Freespace).FreeSpace
      $disk_free_space_gb = [Math]::Round($($disk_free_space/1GB), 2)
      if ($disk_free_space_gb -lt 1) {
        $NotifyIcon.BalloonTipTitle = "$device_id"
        $NotifyIcon.BalloonTipText = "$disk_free_space_gb GB Free"
        $NotifyIcon.BalloonTipIcon = "Warning"
        $NotifyIcon.ShowBalloonTip(5000)
      }
    }

    MainFormEnabled
  }
}

function CheckStatusStopped {
  if ($DataGridView.RowCount -ne 0) {
    MainFormDisabled
    $DataGridView.ClearSelection()
    ForEach ($row in $DataGridView.Rows) {
      if ([bool] (Get-Job -Name $row.Cells[0].Value -ErrorAction Ignore) -eq $false) {
        $row.DefaultCellStyle.BackColor = "#FFFFFF"
        $row.Cells[4].Value = $null
      }
    }
    ForEach ($row in $DataGridView.Rows) {
      if ([bool] (Get-Job -Name $row.Cells[0].Value -ErrorAction Ignore) -eq $false) {
        DisplayModelStatusInDataGrid -row $row
        Start-Sleep -milliseconds $status_check_delay
      }
    }
  MainFormEnabled
  }
}

###################################################################################################################################################################################
#### Get M3U8 #####################################################################################################################################################################
###################################################################################################################################################################################

function GetM3U8 ($site, $model_name) {

  if ($site -eq "Chaturbate") {
    $curlArguments1 = "--silent", "--show-error", "--insecure", 
                      "--user-agent", "$user_agent", 
                      "--url", "$url_chaturbate/api/chatvideocontext/$model_name/"
    $json = $null
    $json = & $curl_exe @curlArguments1 | ConvertFrom-Json
    $m3u8 = $json.hls_source
    if ($to_use_proxy_list -eq 1 -and ($json.detail -match "region" -or $json -eq $null)) {
      if (Test-Path $proxy_list_path) {
        $proxy_list = @(Get-Content -Path $proxy_list_path)
      }

      $first_proxies_number = 7
      if ($proxy_list.Count -lt $first_proxies_number) {
        $first_proxies_number = $proxy_list.Count
      }

      for ($i = 0; $i -lt $first_proxies_number; $i++) {
        switch ($proxy_protocol) {
          "socks4" {$proxy_with_prefix = "socks4://" + $proxy_list[$i]}
          "socks4a" {$proxy_with_prefix = "socks4a://" + $proxy_list[$i]}
          "socks5" {$proxy_with_prefix = "socks5://" + $proxy_list[$i]}
          "socks5h" {$proxy_with_prefix = "socks5h://" + $proxy_list[$i]}
          Default {$proxy_with_prefix = $proxy_list[$i]}
        }
        Write-Host $proxy_with_prefix
        $curlArguments2 = "--silent", "--show-error", "--insecure", 
                          "--user-agent", "$user_agent", 
                          "--proxy", "$proxy_with_prefix", 
                          "--connect-timeout", "$connect_timeout", 
                          "--url", "$url_chaturbate/api/chatvideocontext/$model_name/"
        $json = & $curl_exe @curlArguments2 | ConvertFrom-Json
        $m3u8 = $json.hls_source
        if ($json) {
          Break
        }
      }

    }
  }

  if ($site -eq "BongaCams") {
    $curlArguments3 = "--silent", "--show-error", "--insecure", 
                      "--user-agent", "$user_agent", 
                      "--header", "X-ab-Split-Group: $x_ab_split_group", 
                      "--header", "Cookie: bonga20120608=$cookie_bonga20120608", 
                      "--header", "X-Requested-With: XMLHttpRequest", 
                      "--data", "method=getRoomData&args%5B%5D=$model_name", 
                      "--url", "$url_bongacams/tools/amf.php"
    $json = $null
    $json = & $curl_exe @curlArguments3 | ConvertFrom-Json
    $view_server = $json.localData.videoServerUrl
    if ($view_server) {
      $m3u8 = "https:" + $view_server + "/hls/stream_" + $username + "/playlist.m3u8"
    }

    if ($to_use_proxy_list -eq 1 -and ($json.status -eq "error" -or $json -eq $null)) {
      if (Test-Path $proxy_list_path) {
        $proxy_list = @(Get-Content -Path $proxy_list_path)
      }

      $first_proxies_number = 7
      if ($proxy_list.Count -lt $first_proxies_number) {
        $first_proxies_number = $proxy_list.Count
      }

      for ($i = 0; $i -lt $first_proxies_number; $i++) {
        switch ($proxy_protocol) {
          "socks4" {$proxy_with_prefix = "socks4://" + $proxy_list[$i]}
          "socks4a" {$proxy_with_prefix = "socks4a://" + $proxy_list[$i]}
          "socks5" {$proxy_with_prefix = "socks5://" + $proxy_list[$i]}
          "socks5h" {$proxy_with_prefix = "socks5h://" + $proxy_list[$i]}
          Default {$proxy_with_prefix = $proxy_list[$i]}
        }
        Write-Host $proxy_with_prefix
        $curlArguments4 = "--silent", "--show-error", "--insecure", 
                          "--user-agent", "$user_agent", 
                          "--header", "X-ab-Split-Group: $x_ab_split_group", 
                          "--header", "Cookie: bonga20120608=$cookie_bonga20120608", 
                          "--header", "X-Requested-With: XMLHttpRequest", 
                          "--data", "method=getRoomData&args%5B%5D=$model_name", 
                          "--proxy", "$proxy_with_prefix", 
                          "--connect-timeout", "$connect_timeout", 
                          "--url", "$url_bongacams/tools/amf.php"

        $json = & $curl_exe @curlArguments4 | ConvertFrom-Json
        $view_server = $json.localData.videoServerUrl
        $m3u8 = "https:" + $view_server + "/hls/stream_" + $model_name + "/playlist.m3u8"
        if ($json) {
          Break
        }
      }

    }
  }

  if ($site -eq "StripChat") {
    $curlArguments5 = "--silent", "--show-error", "--insecure", 
                      "--user-agent", "$user_agent", 
                      "--url", "$url_stripchat/api/front/models/username/$model_name/cam/"
    $json_cam = $null
    $json_cam = & $curl_exe @curlArguments5 | ConvertFrom-Json

    if ($to_use_proxy_list -eq 1 -and $json_cam -eq $null) {
      if (Test-Path $proxy_list_path) {
        $proxy_list = @(Get-Content -Path $proxy_list_path)
      }

      $first_proxies_number = 7
      if ($proxy_list.Count -lt $first_proxies_number) {
        $first_proxies_number = $proxy_list.Count
      }

      for ($i = 0; $i -lt $first_proxies_number; $i++) {
        switch ($proxy_protocol) {
          "socks4" {$proxy_with_prefix = "socks4://" + $proxy_list[$i]}
          "socks4a" {$proxy_with_prefix = "socks4a://" + $proxy_list[$i]}
          "socks5" {$proxy_with_prefix = "socks5://" + $proxy_list[$i]}
          "socks5h" {$proxy_with_prefix = "socks5h://" + $proxy_list[$i]}
          Default {$proxy_with_prefix = $proxy_list[$i]}
        }
        Write-Host $proxy_with_prefix
        $curlArguments7 = "--silent", "--show-error", "--insecure", 
                          "--user-agent", "$user_agent", 
                          "--proxy", "$proxy_with_prefix", 
                          "--connect-timeout", "$connect_timeout", 
                          "--url", "$url_stripchat/api/front/models/username/$model_name/cam/"
        $json_cam = & $curl_exe @curlArguments7 | ConvertFrom-Json
        $view_server = $json_cam.viewServers."flashphoner-hls"
        $curlArguments8 = "--silent", "--show-error", "--insecure", 
                          "--user-agent", "$user_agent", 
                          "--proxy", "$proxy_with_prefix", 
                          "--connect-timeout", "$connect_timeout", 
                          "--url", "$url_stripchat/api/front/users/username/$model_name/"
        $model_id = (& $curl_exe @curlArguments8 | ConvertFrom-Json).user.id
        $m3u8 = "https://b-" + $view_server + ".stripst.com/hls/" + $model_id + "/" + $model_id + ".m3u8"
        if ($json_cam) {
          Break
        }
      }

    }

    $view_server = $json_cam.viewServers."flashphoner-hls"
    $curlArguments6 = "--silent", "--show-error", "--insecure", 
                      "--user-agent", "$user_agent", 
                      "--url", "$url_stripchat/api/front/users/username/$model_name/"
    $model_id = (& $curl_exe @curlArguments6 | ConvertFrom-Json).user.id
    $m3u8 = "https://b-" + $view_server + ".stripst.com/hls/" + $model_id + "/" + $model_id + ".m3u8"
  }

  if ($site -eq "MyFreeCams") {
    [System.Windows.Forms.MessageBox]::Show("Not supported for MyFreeCams", "Streamlink GUI", "OK", "Information")
  }

  return $m3u8

}

#### Watch Stream #################################################################################################################################################################

function WatchStream {
  MainFormDisabled
  $m3u8 = GetM3U8 -site $DataGridView.SelectedCells[2].FormattedValue -model_name $DataGridView.SelectedCells[1].FormattedValue
  if ($m3u8 -match "m3u8") {
    if ($MenuItem_WatchStreamsInVLC.Checked -eq $true) {
      if ($DataGridView.SelectedCells[2].FormattedValue -eq "StripChat") {
        $default_stream = "best"
      } else {
        $default_stream = $DataGridView.SelectedCells[3].FormattedValue
      }
      $model_name = $DataGridView.SelectedCells[1].FormattedValue
      $open_media_player = {
        param ($streamlink_exe, $vlc_media_player_path, $vlc_network_caching, $m3u8, $default_stream, $model_name) 
        $player = '--player=' + $vlc_media_player_path + ' --network-caching=' + $vlc_network_caching
        & $streamlink_exe $player --title "$model_name" --url "$m3u8" --default-stream "$default_stream"
      }
      Start-Process PowerShell -ArgumentList "-WindowStyle Hidden -Command (Invoke-Command -ScriptBlock {$open_media_player} -ArgumentList '$streamlink_exe', '$vlc_media_player_path', '$vlc_network_caching', '$m3u8', '$default_stream', '$model_name')"
    } elseif ($MenuItem_WatchStreamsInBrowser.Checked -eq $true) {
$html = @"
<html><head>
<title>$($DataGridView.SelectedCells[1].FormattedValue)</title>
<link href="https://vjs.zencdn.net/7.6.5/video-js.min.css" rel="stylesheet">
<script src='https://vjs.zencdn.net/7.6.5/video.min.js'></script>
<script src="https://unpkg.com/videojs-contrib-quality-levels@2.0.9/dist/videojs-contrib-quality-levels.min.js"></script>
<script src="https://unpkg.com/videojs-hls-quality-selector@1.0.5/dist/videojs-hls-quality-selector.min.js"></script>
</head><body>
<video id="video" class="video-js" preload="metadata" height="504" autoplay controls muted><source src="$m3u8" type='application/x-mpegURL'></video>
<script>var player = videojs('video');player.hlsQualitySelector();</script>
</body></html>
"@
      $html_page_name = $DataGridView.SelectedCells[1].FormattedValue + ".htm"; $html > $html_page_name; start $html_page_name; Sleep 3; Remove-Item $html_page_name
    }
  }
  MainFormEnabled
}

#### Show Available Streams #######################################################################################################################################################

function ShowAvailableStreams ($site, $model_name) {

  if ($site -eq "Chaturbate") {
    $m3u8 = $null
    $m3u8 = GetM3U8 -site $site -model_name $model_name
    $curlArguments1 = "--silent", "--show-error", "--insecure", 
                      "--user-agent", "$user_agent", 
                      "--url", "$m3u8"
    if ($m3u8) {
      $m3u8_file = & $curl_exe @curlArguments1
    }
    $array = @()
    ForEach ($string in $m3u8_file) {
      if ($string.StartsWith("#EXT-X-STREAM-INF:") -eq $true) {
        $array += $string -replace "#EXT-X-STREAM-INF.*NAME=`"FPS:", "" -replace "`",CODECS=.*RESOLUTION="," FPS, "
        $array += "`n"
      }
    }
    if ($array.Length -ne 0) {
      [System.Windows.Forms.MessageBox]::Show("$array", "$model_name - $site", "OK", "Information")
    } else {
      [System.Windows.Forms.MessageBox]::Show("No streams available", "$model_name - $site", "OK", "Information")
    }
  } 

  if ($site -eq "BongaCams") {
    $m3u8 = $null
    $m3u8 = GetM3U8 -site $site -model_name $model_name
    $curlArguments2 = "--silent", "--show-error", "--insecure", 
                      "--user-agent", "$user_agent", 
                      "--url", "$m3u8"
    if ($m3u8) {
      $m3u8_file = & $curl_exe @curlArguments2
    }
    $array = @()
    ForEach ($string in $m3u8_file) {
      if ($string.StartsWith("#EXT-X-STREAM-INF:") -eq $true) {
        $array += $string -replace "#EXT-X-STREAM-INF.*RESOLUTION=", "" -replace ",CODECS.*", ""
        $array += "`n"
      }
    }
    if ($array.Length -ne 0) {
      [System.Windows.Forms.MessageBox]::Show("$array", "$model_name - $site", "OK", "Information")
    } else {
      [System.Windows.Forms.MessageBox]::Show("No streams available", "$model_name - $site", "OK", "Information")
    }
  }

  if ($site -eq "StripChat") {
    $curlArguments3 = "--silent", "--show-error", "--insecure", 
                      "--user-agent", "$user_agent", 
                      "--url", "$url_stripchat/api/front/models/username/$model_name/cam/"
    $json = & $curl_exe @curlArguments3 | ConvertFrom-Json
    if ($json.isCamAvailable -match "True") {
      $height = $json.broadcastSettings.height
      $resolutions = $json.broadcastSettings.resolutions
      $array = @()
      $array += "$height" + "p`n"
      ForEach ($resolution in $resolutions.PSObject.Properties) {
        $array += $resolution.Name
        $array += "`n"
      }
      [System.Windows.Forms.MessageBox]::Show("$array", "$model_name - $site", "OK", "Information")
    } else {
      [System.Windows.Forms.MessageBox]::Show("No streams available", "$model_name - $site", "OK", "Information")
    }
  }

  if ($site -eq "MyFreeCams") {
    [System.Windows.Forms.MessageBox]::Show("Not supported for MyFreeCams", "Streamlink GUI", "OK", "Information")
  }

}

###################################################################################################################################################################################
#### Last Broadcast ###############################################################################################################################################################
###################################################################################################################################################################################

function ShowLastBroadcastTime ($site, $model_name) {

  if ($site -eq "Chaturbate") {
    $curlArguments1 = "--silent", "--show-error", "--insecure", 
                      "--user-agent", "$user_agent", 
                      "--url", "$url_chaturbate/api/biocontext/$model_name/"
    $json = $null
    $json = & $curl_exe @curlArguments1 | ConvertFrom-Json

    if ($json.detail -notmatch "region") {
      $time_since_last_broadcast = $json.time_since_last_broadcast
      $last_broadcast = $json.last_broadcast
      $last_broadcast_day, $last_broadcast_time = $last_broadcast.Split("T")
      if ($json) {
        [System.Windows.Forms.MessageBox]::Show("Last Broadcast:`n$time_since_last_broadcast`n$last_broadcast_day`n$last_broadcast_time", "$model_name - $site", "OK", "Information")
      }
    }

    if ($to_use_proxy_list -eq 1 -and ($json.detail -match "region" -or $json -eq $null)) {
      if (Test-Path $proxy_list_path) {
        $proxy_list = @(Get-Content -Path $proxy_list_path)
      }

      $first_proxies_number = 7
      if ($proxy_list.Count -lt $first_proxies_number) {
        $first_proxies_number = $proxy_list.Count
      }

      for ($i = 0; $i -lt $first_proxies_number; $i++) {
        switch ($proxy_protocol) {
          "socks4" {$proxy_with_prefix = "socks4://" + $proxy_list[$i]}
          "socks4a" {$proxy_with_prefix = "socks4a://" + $proxy_list[$i]}
          "socks5" {$proxy_with_prefix = "socks5://" + $proxy_list[$i]}
          "socks5h" {$proxy_with_prefix = "socks5h://" + $proxy_list[$i]}
          Default {$proxy_with_prefix = $proxy_list[$i]}
        }
        Write-Host $proxy_with_prefix
        $curlArguments2 = "--silent", "--insecure", 
                          "--user-agent", "$user_agent", 
                          "--proxy", "$proxy_with_prefix", 
                          "--connect-timeout", "$connect_timeout", 
                          "--url", "$url_chaturbate/api/biocontext/$model_name/"
        $json = & $curl_exe @curlArguments2 | ConvertFrom-Json
        if ($json) {
          $time_since_last_broadcast = $json.time_since_last_broadcast
          $last_broadcast = $json.last_broadcast
          $last_broadcast_day, $last_broadcast_time = $last_broadcast.Split("T")
          [System.Windows.Forms.MessageBox]::Show("Last Broadcast:`n$time_since_last_broadcast`n$last_broadcast_day`n$last_broadcast_time", "$model_name - $site", "OK", "Information")
          Break
        }
      }
    }
  }

  if ($site -eq "BongaCams") {
    $curlArguments3 = "--silent", "--show-error", "--location", "--insecure", 
                      "--user-agent", "$user_agent", 
                      "--url", "$url_bongacams/profile/$model_name"
    $profile_page = $null
    $profile_page = & $curl_exe @curlArguments3
    $profile_last_login_string = $null
    $profile_last_login_string = $profile_page | Select-String -Pattern '<div class="last_login">'
    $last_login = $null
    $last_login = $profile_page | Select-String -Pattern '.*' | Where {$_.LineNumber -eq $($profile_last_login_string.LineNumber + 3)}
    $last_login = $last_login -replace "^ *", "" -replace " *</span>", ""
    if ($profile_last_login_string) {
      [System.Windows.Forms.MessageBox]::Show("Last Login:`n$last_login", "$model_name - $site", "OK", "Information")
    } elseif ($to_use_proxy_list -eq 1) {
      if (Test-Path $proxy_list_path) {
        $proxy_list = @(Get-Content -Path $proxy_list_path)
      }

      $first_proxies_number = 7
      if ($proxy_list.Count -lt $first_proxies_number) {
        $first_proxies_number = $proxy_list.Count
      }

      for ($i = 0; $i -lt $first_proxies_number; $i++) {
        switch ($proxy_protocol) {
          "socks4" {$proxy_with_prefix = "socks4://" + $proxy_list[$i]}
          "socks4a" {$proxy_with_prefix = "socks4a://" + $proxy_list[$i]}
          "socks5" {$proxy_with_prefix = "socks5://" + $proxy_list[$i]}
          "socks5h" {$proxy_with_prefix = "socks5h://" + $proxy_list[$i]}
          Default {$proxy_with_prefix = $proxy_list[$i]}
        }
        Write-Host $proxy_with_prefix
        $curlArguments4 = "--silent", "--show-error", "--location", "--insecure", 
                          "--user-agent", "$user_agent", 
                          "--proxy", "$proxy_with_prefix", 
                          "--connect-timeout", "$connect_timeout", 
                          "--url", "$url_bongacams/profile/$model_name"
        $profile_page = & $curl_exe @curlArguments4
        $profile_last_login_string = $null
        $profile_last_login_string = $profile_page | Select-String -Pattern '<div class="last_login">'
        $last_login = $null
        $last_login = $profile_page | Select-String -Pattern '.*' | Where {$_.LineNumber -eq $($profile_last_login_string.LineNumber + 3)}
        $last_login = $last_login -replace "^ *", "" -replace " *</span>", ""
        if ($profile_last_login_string) {
          [System.Windows.Forms.MessageBox]::Show("Last Login:`n$last_login", "$model_name - $site", "OK", "Information")
          Break
        }
      }
    }
  }

  if ($site -eq "StripChat") {
    [System.Windows.Forms.MessageBox]::Show("Not supported for StripChat", "Streamlink GUI", "OK", "Information")
  }

  if ($site -eq "MyFreeCams") {
    [System.Windows.Forms.MessageBox]::Show("Not supported for MyFreeCams", "Streamlink GUI", "OK", "Information")
  }

}

#### Update Status Bar Text #######################################################################################################################################################

function UpdateStatusBarText {
  $StatusBar.Text = "Running: $($(Get-Job).Count)  |  Total:$($DataGridView.RowCount)"
}

#### Copy Name to Clipboard #######################################################################################################################################################

function CopyNameToClipboard {
  Set-Clipboard -Value $DataGridView.SelectedCells[1].FormattedValue
}

#### AutoSave Model List ##########################################################################################################################################################

function AutoSaveModelList {
  if ($autosave_model_list -eq 1) {
    Clear-Content $model_list_path
    ForEach ($row in $DataGridView.Rows) {
      $row.Cells[1].Value + ";" + $row.Cells[2].Value + ";" + $row.Cells[3].Value | Out-File -Append $model_list_path
    }
  }
}

#### Change Quality ###############################################################################################################################################################

function ChangeQuality {
  if ($DataGridView.SelectedRows.Count -eq 1) {
    MainFormDisabled
    if ($TextBox_OtherQuality.Enabled -eq $false -or ($TextBox_OtherQuality.Enabled -eq $true -and -not [string]::IsNullOrEmpty($TextBox_OtherQuality.Text))) {
      if ($ComboBox_VideoQuality.SelectedItem -eq "other") {
        $selected_quality = $TextBox_OtherQuality.Text
      } else {
        $selected_quality = $ComboBox_VideoQuality.SelectedItem
      }
    }
    if ([bool] (Get-Job -Name $DataGridView.SelectedCells[0].Value -ErrorAction Ignore) -eq $false -and $DataGridView.SelectedCells[3].Value -ne $selected_quality) {
      $DataGridView.SelectedCells[3].Value = $selected_quality
      $ComboBox_VideoQuality.SelectedIndex = 0
      $DataGridView.ClearSelection()
      
    }
  AutoSaveModelList
  MainFormEnabled
 }
}

#### Change Quality And Restart ###################################################################################################################################################

function ChangeQualityAndRestart {
  if ($DataGridView.SelectedRows.Count -eq 1) {
    MainFormDisabled
    if ($TextBox_OtherQuality.Enabled -eq $false -or ($TextBox_OtherQuality.Enabled -eq $true -and -not [string]::IsNullOrEmpty($TextBox_OtherQuality.Text))) {
      if ($ComboBox_VideoQuality.SelectedItem -eq "other") {
        $selected_quality = $TextBox_OtherQuality.Text
      } else {
        $selected_quality = $ComboBox_VideoQuality.SelectedItem
      }
    }
    ForEach ($row in $DataGridView.SelectedRows) {
      if ([bool] (Get-Job -Name $row.Cells[0].Value -ErrorAction Ignore) -eq $true -and $row.Cells[3].Value -ne $selected_quality) {
        Stop-Job -Name $row.Cells[0].Value
        Remove-Job -Name $row.Cells[0].Value -Force
        $row.Cells[5].Value = $null
        $row.Cells[3].Value = $selected_quality
        if ($check_status_after_start -eq 1) {
          DisplayModelStatusInDataGrid -row $row
        }
        if ([bool] (Get-Job -Name $row.Cells[0].Value -ErrorAction Ignore) -eq $false) {
          StartRecording -job_name $row.Cells[0].Value -site $row.Cells[2].Value -model_name $row.Cells[1].Value -stream_quality $row.Cells[3].Value
          $row.Cells[5].Value = $rec_icon
          $ComboBox_VideoQuality.SelectedIndex = 0
          $DataGridView.ClearSelection()
        }
      }
    }
    AutoSaveModelList
    MainFormEnabled
 }
}

#### Add New Row ##################################################################################################################################################################

$global:i = $DataGridView.RowCount + 1

function DisplayRowInDataGrid ($model_name, $site, $quality) {

  if ($global:i -lt 10) {
    $row_number = "000" + $global:i++
  } elseif ($global:i -lt 100) {
    $row_number = "00" + $global:i++
  } elseif ($global:i -lt 1000) {
    $row_number = "0" + $global:i++
  } else {
    $row_number = $global:i++
  }

  $DataGridView.Rows.Add("$row_number", "$model_name", "$site", "$quality", $null, $null) | Out-Null

  if ($DataGridView.Height -ge $DataGridView.MaximumSize.Height) {
    $MainForm.Height = $DataGridView.MaximumSize.Height + 165
  }

  AutoSaveModelList

}

function DisplayRowInDataGridAndStartRecording ($model_name, $site, $quality) {

  if ($global:i -lt 10) {
    $row_number = "000" + $global:i++
  } elseif ($global:i -lt 100) {
    $row_number = "00" + $global:i++
  } elseif ($global:i -lt 1000) {
    $row_number = "0" + $global:i++
  } else {
    $row_number = $global:i++
  }

  $NewRow = New-Object System.Windows.Forms.DataGridViewRow
  $NewRow.CreateCells($DataGridView, "$row_number", "$model_name", "$site", "$quality", $null, $null)
  $DataGridView.Rows.Add($NewRow) | Out-Null

  if ($DataGridView.Height -ge $DataGridView.MaximumSize.Height) {
    $MainForm.Height = $DataGridView.MaximumSize.Height + 165
  }

  AutoSaveModelList

  StartRecording -job_name $row_number -site "$site" -model_name $model_name -stream_quality $quality
  $NewRow.Cells[5].Value = $rec_icon
  if ($check_status_after_start -eq 1) {
    DisplayModelStatusInDataGrid -row $NewRow
  }
  $DataGridView.ClearSelection()

}

function AddNewRow {
  if (-not [string]::IsNullOrEmpty($TextBox_ModelName.Text)) {

    if ($TextBox_OtherQuality.Enabled -eq $true -and [string]::IsNullOrEmpty($TextBox_OtherQuality.Text)) {
      [System.Windows.Forms.MessageBox]::Show("Please enter video quality", "Error", "OK", "Exclamation")
    }

    if ($TextBox_OtherQuality.Enabled -eq $false -or ($TextBox_OtherQuality.Enabled -eq $true -and -not [string]::IsNullOrEmpty($TextBox_OtherQuality.Text))) {
      if ($ComboBox_VideoQuality.SelectedItem -eq "other") {
        $selected_quality = $TextBox_OtherQuality.Text
      } else {
        $selected_quality = $ComboBox_VideoQuality.SelectedItem
      }

      if ($TextBox_ModelName.Text -notmatch "/") {
        ForEach ($row in $DataGridView.Rows) {
          if ($TextBox_ModelName.Text -eq $row.Cells[1].Value -and $ComboBox_SiteList.SelectedItem -eq $row.Cells[2].Value -and $selected_quality -eq $row.Cells[3].Value) {
            [System.Windows.Forms.MessageBox]::Show("$($TextBox_ModelName.Text) already in the list!", "Error", "OK", "Exclamation")
            $TextBox_ModelName.Clear()
            return
          }
        }

        if ($ComboBox_SiteList.SelectedItem -eq "Chaturbate") {
          $ModelName = $TextBox_ModelName.Text.ToLower()
        } else {
          $ModelName = $TextBox_ModelName.Text
        }

        if ($MenuItem_StartRecAfterAdd.Checked -eq $false) {
          DisplayRowInDataGrid -model_name $ModelName -site $ComboBox_SiteList.SelectedItem -quality $selected_quality
        } else {
          DisplayRowInDataGridAndStartRecording -model_name $ModelName -site $ComboBox_SiteList.SelectedItem -quality $selected_quality
        }

      } elseif ($TextBox_ModelName.Text -notmatch "/profile") {
        if ($TextBox_ModelName.Text -match "https|http") {
          $Site = $TextBox_ModelName.Text.Split("/")[2]
          $ModelName = $TextBox_ModelName.Text.Split("/")[3]
        } else {
          $Site = $TextBox_ModelName.Text.Split("/")[0]
          $ModelName = $TextBox_ModelName.Text.Split("/")[1]
        }
        switch -regex ($Site) {
          "chaturbate" {$Site = "Chaturbate"}
          "bongacams" {$Site = "BongaCams"}
          "stripchat" {$Site = "StripChat"}
          "myfreecams" {$Site = "MyFreeCams"}
          Default {$Site = $null}
        }
        if (-not [string]::IsNullOrEmpty($Site) -and -not [string]::IsNullOrEmpty($ModelName)) {
          ForEach ($row in $DataGridView.Rows) {
            if ($ModelName -eq $row.Cells[1].Value -and $Site -eq $row.Cells[2].Value -and $selected_quality -eq $row.Cells[3].Value) {
              [System.Windows.Forms.MessageBox]::Show("$ModelName already in the list!", "Error", "OK", "Exclamation")
              $TextBox_ModelName.Clear()
              return
            }
          }

        if ($Site -eq "Chaturbate") {
          $ModelName = $ModelName.ToLower()
        }

        if ($Site -eq "MyFreeCams") {
          $ModelName = $ModelName -replace "^#", ""
        }

        if ($MenuItem_StartRecAfterAdd.Checked -eq $false) {
          DisplayRowInDataGrid -model_name $ModelName -site $Site -quality $selected_quality
        } else {
          DisplayRowInDataGridAndStartRecording -model_name $ModelName -site $Site -quality $selected_quality
        }

        }
      }
      $TextBox_ModelName.Clear()
      $ComboBox_VideoQuality.SelectedIndex = 0
      $TextBox_ModelName.Focus()
      UpdateStatusBarText
    }
  }
}

#### Start / Stop / Remove ########################################################################################################################################################

function StartAll {
  if ($DataGridView.RowCount -ne 0) {
    MainFormDisabled
    ForEach ($row in $DataGridView.Rows) {
      if ([bool] (Get-Job -Name $row.Cells[0].Value -ErrorAction Ignore) -eq $false) {
        StartRecording -job_name $row.Cells[0].Value -site $row.Cells[2].Value -model_name $row.Cells[1].Value -stream_quality $row.Cells[3].Value
        $row.Cells[5].Value = $rec_icon
        Start-Sleep -milliseconds $start_rec_delay
      }
    }
    if ($check_status_after_start -eq 1) {
      ForEach ($row in $DataGridView.Rows) {
        DisplayModelStatusInDataGrid -row $row
      }
    }
    $DataGridView.ClearSelection()
    UpdateStatusBarText
    MainFormEnabled
  }
}

function StartSelected {
  MainFormDisabled
  ForEach ($row in $DataGridView.SelectedRows) {
    if ([bool] (Get-Job -Name $row.Cells[0].Value -ErrorAction Ignore) -eq $false) {
      StartRecording -job_name $row.Cells[0].Value -site $row.Cells[2].Value -model_name $row.Cells[1].Value -stream_quality $row.Cells[3].Value
      $row.Cells[5].Value = $rec_icon
      if ($DataGridView.SelectedRows.Count -gt 1) {
        Start-Sleep -milliseconds $start_rec_delay
      }
    }
    if ($check_status_after_start -eq 1) {
      DisplayModelStatusInDataGrid -row $row
    }
  }
  $DataGridView.ClearSelection()
  UpdateStatusBarText
  MainFormEnabled
}

function StopAll {
  MainFormDisabled
  Get-Job | Stop-Job
  Get-Job | Remove-Job -Force
  ForEach ($row in $DataGridView.Rows) {
    $row.Cells[5].Value = $null
  }
  UpdateStatusBarText
  MainFormEnabled
}

function StopSelected {
  MainFormDisabled
  ForEach ($row in $DataGridView.SelectedRows) {
    if ([bool] (Get-Job -Name $row.Cells[0].Value -ErrorAction Ignore) -eq $true) {
      Stop-Job -Name $row.Cells[0].Value
      Remove-Job -Name $row.Cells[0].Value -Force
      $row.Cells[5].Value = $null
    }
  }
  $DataGridView.ClearSelection()
  UpdateStatusBarText
  MainFormEnabled
}

function StopAllAndClear {
  MainFormDisabled
  Get-Job | Stop-Job
  Get-Job | Remove-Job -Force
  $global:i = 1
  $DataGridView.Rows.Clear()
  UpdateStatusBarText
  MainFormEnabled
}

function StopAllAndExit {
  if ($DataGridView.RowCount -ne 0 -and $autosave_model_list -ne 1) {
    $msgBoxInput = [System.Windows.Forms.MessageBox]::Show("Save current list?", "Question", "YesNoCancel", "Question")
    switch  ($msgBoxInput) {
      "Yes" {
        SaveModelList
        Get-Job | Stop-Job
        Get-Job | Remove-Job -Force
        $DataGridView.Rows.Clear()
        $MainForm.Close()
      }
      "No" {
        Get-Job | Stop-Job
        Get-Job | Remove-Job -Force
        $DataGridView.Rows.Clear()
        $MainForm.Close()
      }
      "Cancel" {
        $TextBox_ModelName.Focus()
      }
    }
  } else {
    Get-Job | Stop-Job
    Get-Job | Remove-Job -Force
    $DataGridView.Rows.Clear()
    $MainForm.Close()
  }
}

function RemoveSelected {
  MainFormDisabled
  ForEach ($row in $DataGridView.SelectedRows) {
    if ([bool] (Get-Job -Name $row.Cells[0].Value -ErrorAction Ignore) -eq $true) {
      Stop-Job -Name $row.Cells[0].Value
      Remove-Job -Name $row.Cells[0].Value -Force
    }
    $DataGridView.Rows.Remove($row)
  }
  UpdateStatusBarText
  AutoSaveModelList
  MainFormEnabled
}

#### Move Row #####################################################################################################################################################################

function MoveRow ($direction) {
  if ($DataGridView.SelectedRows.Count -eq 1) {
    $current_row_index = $DataGridView.SelectedRows[0].Index
    $current_row = $DataGridView.Rows[$current_row_index]
    $new_row = $DataGridView.Rows[$current_row_index + $direction]

    if (($current_row_index -eq 0) -and ($direction -eq -1)) {
      return
    }
    elseif (($current_row_index -eq ($DataGridView.Rows.Count - 1)) -and ($direction -eq 1)) {
      return
    }
    else {
      $current_row_color = $current_row.DefaultCellStyle.BackColor
      $new_row_color = $new_row.DefaultCellStyle.BackColor

      for ($i = 0; $i -lt $current_row.Cells.Count; $i++){
        $cell = $current_row.Cells[$i].Value
        $current_row.Cells[$i].Value = $new_row.Cells[$i].Value
        $new_row.Cells[$i].Value = $cell
      }

      $current_row.DefaultCellStyle.BackColor = $new_row_color
      $new_row.DefaultCellStyle.BackColor = $current_row_color
      $current_row.Selected = $false
      $new_row.Selected = $true
      AutoSaveModelList
    }
  }
}

#### Open / Save ##################################################################################################################################################################

function OpenModelList {
  $OpenFileDialog = New-Object System.Windows.Forms.OpenFileDialog
  $OpenFileDialog.InitialDirectory = $InitialDirectory
  $OpenFileDialog.ShowHelp = $true
  $OpenFileDialog.Filter = "Text files (*.txt)|*.txt|All files (*.*)|*.*"
  if ($OpenFileDialog.ShowDialog() -eq "OK") {
    $model_list = @(Get-Content -Path $OpenFileDialog.FileName)
    if ($DataGridView.RowCount -ne 0) {
      $msgBoxInput = [System.Windows.Forms.MessageBox]::Show("Clear current list?", "Question", "YesNo", "Question")
      if ($msgBoxInput -eq "Yes") {
        StopAllAndClear
      }
    }
    ForEach ($string in $model_list) {
      $n, $s, $q = $string.Split(";")
      DisplayRowInDataGrid -model_name $n -site $s -quality $q
    }
  }
  UpdateStatusBarText
}

function SaveModelList {
  if ($DataGridView.RowCount -ne 0) {
    $SaveFileDialog = New-Object System.Windows.Forms.SaveFileDialog
    $SaveFileDialog.InitialDirectory = $InitialDirectory
    $SaveFileDialog.ShowHelp = $true
    $SaveFileDialog.FileName = "model_list.txt"
    $SaveFileDialog.Filter = "Text files (*.txt)|*.txt"
    $SaveFileDialog.ShowDialog() | Out-Null
    if (Test-Path $SaveFileDialog.FileName) {
      Clear-Content $SaveFileDialog.FileName
    }
    ForEach ($row in $DataGridView.Rows) {
      $row.Cells[1].Value + ";" + $row.Cells[2].Value + ";" + $row.Cells[3].Value | Out-File -Append $SaveFileDialog.FileName
    }
  }
}

function OpenModelRoomInBrowser ($site, $model_name) {
  switch  ($site) {
    "Chaturbate" {start "$url_chaturbate/$model_name/"}
    "BongaCams" {start "$url_bongacams/$model_name"}
    "StripChat" {start "$url_stripchat/$model_name"}
    "MyFreeCams" {start "https://www.myfreecams.com/#$model_name"}
  }
}

function OpenRecordsFolder {
  if (Test-Path $save_directory) {
    if ($open_records_folder_in_totalcmd -eq 1) {
      & $totalcmd_path /O /L=$save_directory
    } else {
      Invoke-Item $save_directory
    }
  } else {
    [System.Windows.Forms.MessageBox]::Show("Folder $save_directory not found", "Information", "OK", "Information")
  }
}

#### Open Model List In Browser (Chaturbate) ######################################################################################################################################

function OpenModelListInBrowserCB {
  if ($MenuItem_CB_Navigation2.Text -eq "All") {
    switch  ($MenuItem_CB_Navigation1.Text) {
      "Featured" {$url = "$url_chaturbate/?page=$($MenuItem_CB_Page.Text)"}
      "Female" {$url = "$url_chaturbate/female-cams/?page=$($MenuItem_CB_Page.Text)"}
      "Male" {$url = "$url_chaturbate/male-cams/?page=$($MenuItem_CB_Page.Text)"}
      "Couple" {$url = "$url_chaturbate/couple-cams/?page=$($MenuItem_CB_Page.Text)"}
      "Trans" {$url = "$url_chaturbate/trans-cams/?page=$($MenuItem_CB_Page.Text)"}
    }
  } else {
    switch  ($MenuItem_CB_Navigation1.Text) {
      "Featured" {$nav = "?page=$($MenuItem_CB_Page.Text)"}
      "Female" {$nav = "female/?page=$($MenuItem_CB_Page.Text)"}
      "Male" {$nav = "male/?page=$($MenuItem_CB_Page.Text)"}
      "Couple" {$nav = "couple/?page=$($MenuItem_CB_Page.Text)"}
      "Trans" {$nav = "trans/?page=$($MenuItem_CB_Page.Text)"}
    }
    switch  ($MenuItem_CB_Navigation2.Text) {
      "All"  {$url = "$url_chaturbate"}
      "Teen"  {$url = "$url_chaturbate/teen-cams/$nav"}
      "18 to 21"  {$url = "$url_chaturbate/18to21-cams/$nav"}
      "20 to 30"  {$url = "$url_chaturbate/20to30-cams/$nav"}
      "30 to 50"  {$url = "$url_chaturbate/30to50-cams/$nav"}
      "Mature"  {$url = "$url_chaturbate/mature-cams/$nav"}
      "North American"  {$url = "$url_chaturbate/north-american-cams/$nav"}
      "Other Region"  {$url = "$url_chaturbate/other-region-cams/$nav"}
      "Euro Russian"  {$url = "$url_chaturbate/euro-russian-cams/$nav"}
      "Asian"  {$url = "$url_chaturbate/asian-cams/$nav"}
      "South American"  {$url = "$url_chaturbate/south-american-cams/$nav"}
      "Exhibitionist"  {$url = "$url_chaturbate/exhibitionist-cams/$nav"}
      "HD"  {$url = "$url_chaturbate/hd-cams/$nav"}
      "New" {$url = "$url_chaturbate/new-cams/$nav"}
    }
  }

  if ($MenuItem_CB_ThroughProxy.Checked -eq $true) {
    if (Test-Path $proxy_list_path) {
      $proxy_list = @(Get-Content -Path $proxy_list_path)
    }

      $first_proxies_number = 7
      if ($proxy_list.Count -lt $first_proxies_number) {
        $first_proxies_number = $proxy_list.Count
      }

      for ($i = 0; $i -lt $first_proxies_number; $i++) {
        switch ($proxy_protocol) {
          "socks4" {$proxy_with_prefix = "socks4://" + $proxy_list[$i]}
          "socks4a" {$proxy_with_prefix = "socks4a://" + $proxy_list[$i]}
          "socks5" {$proxy_with_prefix = "socks5://" + $proxy_list[$i]}
          "socks5h" {$proxy_with_prefix = "socks5h://" + $proxy_list[$i]}
          Default {$proxy_with_prefix = $proxy_list[$i]}
      }
      Write-Host $proxy_with_prefix
      $curlArguments = "--silent", "--show-error", "--insecure", 
                       "--user-agent", "$user_agent", 
                       "--proxy", "$proxy_with_prefix", 
                       "--connect-timeout", "$connect_timeout", 
                       "--url", "$url"
      $html_page = $null
      $html_page = & $curl_exe @curlArguments
      if ($html_page) {
        $strings_model_name = $html_page | Where {$_ -match 'data-slug'}
        $models_list = @()
        ForEach ($string in $strings_model_name) {
          $models_list += $string.Split("`"")[9]
        }

        $strings_thumbnail_label = $html_page | Where {$_ -match '<div class="thumbnail_label'}
        $labels_list = @()
        ForEach ($string in $strings_thumbnail_label) {
          $labels_list += $string.Split(">")[1] -replace "</div", ""
        }

        $strings_gender = $html_page | Where {$_ -match '<span class="age'}
        $gender_list = @()
        ForEach ($string in $strings_gender) {
          $gender_list += $string.Split("`"")[1] -replace "age ", ""
        }

        $strings_location = $html_page | Where {$_ -match '<li class="location"'}
        $location_list = @()
        ForEach ($string in $strings_location) {
          $location_list += $string.Split(">")[1] -replace "</li", ""
        }

        $strings_viewers_number = $html_page | Where {$_ -match '<li class="cams">'}
        $viewers_list = @()
        ForEach ($string in $strings_viewers_number) {
          $viewers_list += $string.Split(",")[1] -replace "viewers</li>", "" -replace " ", ""
        }
      }

      if ($models_list) {
        Break
      }

    }

$html = $null
$html = @"
<html><head><meta charset="utf-8"><title>Chaturbate - Page $($MenuItem_CB_Page.Text) ($($models_list.Count)) - $($MenuItem_CB_Navigation1.Text) - $($MenuItem_CB_Navigation2.Text)</title>
<style type="text/css">
.room {position: relative;background: #f0f1f1;width: 360px;float: left;margin: 7px;border: 1px solid #acacac;border-radius: 4px;text-align: center;}
.thumb {height: 270px;}
.title {height: 25px;float: left;color: #0A5A83;font-size: 20px;white-space: pre;}
img {max-height: 100%;}
.label_new {position: absolute;right: 0;padding: 1px 3px;background-color: #71b404;opacity: 0.8;color: #ffffff;text-align: center;font: 16px ubunturegular, Arial, Helvetica, sans-serif}
.label_close {position: absolute;padding: 1px 3px;background-color: #ffffff;opacity: 0.8;color: #000000;text-align: center;font: 16px ubunturegular, Arial, Helvetica, sans-serif}
.save {position: fixed;background: #f0f1f1;color: #0A5A83;font-size: 18px;top: 0;right: 0;padding: 1px 3px;z-index: 100;}
a {text-decoration: none;cursor: pointer;}
</style>
<script>
var interval;function over(x) {interval = setInterval(function(){x.src = 'https://cbjpeg.stream.highwebmedia.com/minifap/' + x.id + '.jpg?' + Math.random();},$cb_thum_animation_speed);}
function out(x) {x.src = 'https://cbjpeg.stream.highwebmedia.com/minifap/' + x.id + '.jpg?' + Math.random();clearInterval(interval);}
window.onload = function() {let cb_blackList = JSON.parse(localStorage.getItem('cb_blackList'));if (cb_blackList) {for (let username of cb_blackList) {
let div = document.getElementById('div_' + username);if (div) {div.style.display = 'none';}}}}
function saveBlackList() {let cb_blackList = localStorage.getItem('cb_blackList');let file = new Blob([cb_blackList], {type: 'txt'});let a = document.createElement('a');let url = URL.createObjectURL(file);
a.href = url;a.download = 'cb_blackList';document.body.appendChild(a);a.click();setTimeout(function() {document.body.removeChild(a);window.URL.revokeObjectURL(url);}, 0);}
</script>
</head><body><div class="save"><a onclick="saveBlackList()">SAVE BLACKLIST</a></div>
"@

for ($i = 0; $i -lt $models_list.Count; $i++) {

  if ($($labels_list[$i]) -eq "NEW") {
    $is_new = '<div class="label_new">NEW</div>'
  } else {
    $is_new = $null
  }

  if ($($labels_list[$i]) -match "CHATURBATING|EXHIBITIONIST|NEW") {
    $label = $null
  } else {
    $label = $($labels_list[$i])
  }

  $hide = $false
  if ($MenuItem_CB_HideMale.Checked -eq $true) {
    if ($($gender_list[$i]) -eq "genderm") {
      $hide = $true
    }
  }
  if ($MenuItem_CB_HideTrans.Checked -eq $true) {
    if ($($gender_list[$i]) -eq "genders") {
      $hide = $true
    }
  }
  if ($MenuItem_CB_HideColombia.Checked -eq $true) {
    if ($($location_list[$i]) -match "colombia|departamento|bogota|medellin|antioquia|santander|risaralda") {
      $hide = $true
    }
  }

if ($hide -eq $false) {
$html += @"
<div id="div_$($models_list[$i])" class="room">
<div class="label_close"><a title = "Hide $($models_list[$i])" onclick="document.getElementById('div_$($models_list[$i])').style.display = 'none';let cb_blackList = JSON.parse(localStorage.getItem('cb_blackList'));if (!cb_blackList) {cb_blackList = []};cb_blackList.push('$($models_list[$i])');localStorage.setItem('cb_blackList', JSON.stringify(cb_blackList));">X</a></div>
$is_new
<div class="thumb"><a href="$url_chaturbate/$($models_list[$i])/" target="_blank"><img src="https://roomimg.stream.highwebmedia.com/ri/$($models_list[$i]).jpg" title="$($location_list[$i])" id="$($models_list[$i])" onmouseover="over(this)" onmouseout="out(this)"></a></div>
<div class="title" style="width: 20%; text-align: left;"> $($viewers_list[$i])</div>
<div class="title" style="width: 60%; text-align: center;">$($models_list[$i])</div>
<div class="title" style="width: 20%; text-align: right;">$label </div>
</div>
"@
}
}
$html > Chaturbate.htm; start Chaturbate.htm; Sleep 3; Remove-Item Chaturbate.htm

  } else {
    Start $url
  }

}

#### Open Model List In Browser (BongaCams) #######################################################################################################################################

function OpenModelListInBrowserBC {
  switch  ($MenuItem_BC_SortBy.Text) {
    "Camscore" {$sort_by = "camscore"}
    "Most Popular rooms" {$sort_by = "popular"}
    "Just logged on" {$sort_by = "logged"}
    "New Models" {$sort_by = "new"}
    "Lovers" {$sort_by = "lovers"}
  }
  switch  ($MenuItem_BC_Navigation.Text) {
    "Females" {$url = "$url_bongacams/tools/listing_v3.php?livetab=female&_include_online_list=1&model_search%5Bbase_sort%5D=$sort_by"}
    "Couples" {$url = "$url_bongacams/tools/listing_v3.php?livetab=couples&_include_online_list=1&model_search%5Bbase_sort%5D=$sort_by"}
    "Males" {$url = "$url_bongacams/tools/listing_v3.php?livetab=male&_include_online_list=1&model_search%5Bbase_sort%5D=$sort_by"}
    "Transsexuals" {$url = "$url_bongacams/tools/listing_v3.php?livetab=transsexual&_include_online_list=1&model_search%5Bbase_sort%5D=$sort_by"}
    "New" {$url = "$url_bongacams/tools/listing_v3.php?livetab=new&_include_online_list=1&model_search%5Bbase_sort%5D=$sort_by"}
  }

  if ($MenuItem_BC_ThroughProxy.Checked -ne $true) {
    $curlArguments = "--silent", "--show-error", "--insecure", 
                     "--user-agent", "$user_agent", 
                     "--header", "X-Requested-With: XMLHttpRequest", 
                     "--url", "$url"
    $json = $null
    $json = & $curl_exe @curlArguments | ConvertFrom-Json
  } elseif ($MenuItem_BC_ThroughProxy.Checked -eq $true) {
    if (Test-Path $proxy_list_path) {
      $proxy_list = @(Get-Content -Path $proxy_list_path)
    }

      $first_proxies_number = 7
      if ($proxy_list.Count -lt $first_proxies_number) {
        $first_proxies_number = $proxy_list.Count
      }

      for ($i = 0; $i -lt $first_proxies_number; $i++) {
        switch ($proxy_protocol) {
          "socks4" {$proxy_with_prefix = "socks4://" + $proxy_list[$i]}
          "socks4a" {$proxy_with_prefix = "socks4a://" + $proxy_list[$i]}
          "socks5" {$proxy_with_prefix = "socks5://" + $proxy_list[$i]}
          "socks5h" {$proxy_with_prefix = "socks5h://" + $proxy_list[$i]}
          Default {$proxy_with_prefix = $proxy_list[$i]}
      }
      Write-Host $proxy_with_prefix
      $curlArguments2 = "--silent", "--show-error", "--insecure", 
                        "--user-agent", "$user_agent", 
                        "--header", "X-Requested-With: XMLHttpRequest", 
                        "--proxy", "$proxy_with_prefix", 
                        "--connect-timeout", "$connect_timeout", 
                        "--url", "$url"
      $json = $null
      $json = & $curl_exe @curlArguments2 | ConvertFrom-Json
      if ($json) {
        Break
      }
    }
  }

if ($json) {
$html = $null
$html = @"
<html><head><meta charset="utf-8"><title>BongaCams - $($MenuItem_BC_Navigation.Text) ($($json.online_models.Count))</title>
<style type="text/css">
.room {position: relative;background: #f0f1f1;width: 480px;float: left;margin: 7px;border: 1px solid #acacac;border-radius: 4px;text-align: center;}
.thumb {height: 270px;}
.title {height: 25px;float: left;color: #0A5A83;font-size: 20px;white-space: pre;}
img {max-height: 100%;}
.label_new {position: absolute;right: 0;padding: 1px 3px;background-color: #71b404;opacity: 0.8;color: #ffffff;text-align: center;font: 16px ubunturegular, Arial, Helvetica, sans-serif}
.label_close {position: absolute;padding: 1px 3px;background-color: #ffffff;opacity: 0.8;color: #000000;text-align: center;font: 16px ubunturegular, Arial, Helvetica, sans-serif}
.save {position: fixed;background: #f0f1f1;color: #0A5A83;font-size: 18px;top: 0;right: 0;padding: 1px 3px;z-index: 100;}
a {text-decoration: none;cursor: pointer;}
</style>
<script>
var interval;function over(x) {interval = setInterval(function() {x.src = 'https://mobile-edge' + x.dataset.vsid + '.bcvcdn.com/stream_' + x.id + '.jpg?' + Math.random();},$bc_thum_animation_speed);}
function out(x) {x.src = 'https://mobile-edge' + x.dataset.vsid + '.bcvcdn.com/stream_' + x.id + '.jpg?' + Math.random();clearInterval(interval);}
window.onload = function() {let bc_blackList = JSON.parse(localStorage.getItem('bc_blackList'));if (bc_blackList) {for (let username of bc_blackList) {
let div = document.getElementById('div_' + username);if (div) {div.style.display = 'none';}}}}
function saveBlackList() {let bc_blackList = localStorage.getItem('bc_blackList');let file = new Blob([bc_blackList], {type: 'txt'});let a = document.createElement('a');let url = URL.createObjectURL(file);
a.href = url;a.download = 'bc_blackList';document.body.appendChild(a);a.click();setTimeout(function() {document.body.removeChild(a);window.URL.revokeObjectURL(url);}, 0);}
</script>
</head><body><div class="save"><a onclick="saveBlackList()">SAVE BLACKLIST</a></div>
"@

for ($i = 0; $i -lt $json.online_models.Count; $i++) {
$model = $null
$model = $json.online_models[$i] | Where {$_.room -eq "public" -and $_.is_away -match "False" -and $_.viewers -gt $MenuItem_BC_MinViewers.Text}
if ($model) {
  if ($($model.is_new) -eq 1) {
    $is_new = '<div class="label_new">NEW</div>'
  } else {
    $is_new = $null
  }
$html += @"
<div id="div_$($model.username)" class="room">
<div class="label_close"><a title = "Hide $($model.username)" onclick="document.getElementById('div_$($model.username)').style.display = 'none';let bc_blackList = JSON.parse(localStorage.getItem('bc_blackList'));if (!bc_blackList) {bc_blackList = []};bc_blackList.push('$($model.username)');localStorage.setItem('bc_blackList', JSON.stringify(bc_blackList));">X</a></div>
$is_new
<div class="thumb"><a href="$url_bongacams/$($model.username)" target="_blank"><img src="https://mobile-edge$($model.vsid).bcvcdn.com/stream_$($model.username).jpg" id="$($model.username)" data-vsid="$($model.vsid)" onmouseover="over(this)" onmouseout="out(this)"></a></div>
<div class="title" style="width: 20%; text-align: left;"> $($model.viewers)</div>
<div class="title" style="width: 60%; text-align: center">$($model.username)</div>
<div class="title" style="width: 20%; text-align: right">$($model.vq) </div>
</div>
"@
}
}
$html > BongaCams.htm; start BongaCams.htm; Sleep 3; Remove-Item BongaCams.htm
}
}

#### Show Balloon Tip #############################################################################################################################################################

function ShowBalloonTip ($site, $name, $status) {
 if ($show_notify_icon -eq 1) {
   $NotifyIcon.BalloonTipTitle = $site
   $NotifyIcon.BalloonTipText = "$name $status"
   $NotifyIcon.BalloonTipIcon = "Info"
   $NotifyIcon.ShowBalloonTip(5000)
 }
}

#### Icons ########################################################################################################################################################################

function Base64toIcon ($base64) {
  $stream = [System.IO.MemoryStream][System.Convert]::FromBase64String($base64)
  $bmp = [System.Drawing.Bitmap]::FromStream($stream)
  $handle = $bmp.GetHicon()
  $icon = [System.Drawing.Icon]::FromHandle($handle)
  return $icon
}

$main_icon = Base64toIcon -base64 "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"
$waiting_icon = Base64toIcon -base64 "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"
$main_icon_green = Base64toIcon -base64 "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"
$waiting_icon_green = Base64toIcon -base64 "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"
$rec_icon = Base64toIcon -base64 "iVBORw0KGgoAAAANSUhEUgAAAA0AAAANCAYAAABy6+R8AAAAIGNIUk0AAHolAACAgwAA+f8AAIDoAABSCAABFVgAADqXAAAXb9daH5AAAAEaSURBVHjanJIxjsIwEEU/ewKnSozkNrvS1hFcgIa9Ua6B7XCrUAHbJVKqGEQRpXCav4UhShAN+6XRyNa84v+ZBUm8rSfoC4CNouiklLpGUXQCsAfwPZufQD9JkjhjDOu6ph8G1nVNrTWTJHEAts/Q53K5vB0OB75SWZaUUt4ApFNoXxRFmOh7Ms/J9Tr0vidJWmsJoBghIcRv0zQByvPw/ag8J0k2TUMhxHmElFIXPwwByrI5lGUkSe89lVJXkvgAgK7rrq5tQxybzTze+9s5h67r3DTywlo797RazTwZYwjAToNIpZS3sizfSg8AtnEct1rrsCfvWVUVd7sd4zhuX+3poRSAFkIclVIXIcQRgLlfymhn8Z/b+xsAQP8zwAj1iroAAAAASUVORK5CYII="
$open_folder_icon = Base64toIcon -base64 "iVBORw0KGgoAAAANSUhEUgAAABAAAAAQCAYAAAAf8/9hAAAABmJLR0QA/wD/AP+gvaeTAAAAaklEQVQ4je3SMQqDQBCF4Q8RO5s9s6Qwt8hJvI1XyFpoiiwRrXYNgoU/DAwD75/mcRUeiFgwY0BbIogIaa/wStJslsKJ6PaCEgLe/wh+mepAcMMtWAWTtUg5BN8uqNPhiRFNpmBCX/DwRD5abR0ae7y3NwAAAABJRU5ErkJggg=="
$timer_icon = Base64toIcon -base64 "iVBORw0KGgoAAAANSUhEUgAAABAAAAAQCAYAAAAf8/9hAAAABmJLR0QA/wD/AP+gvaeTAAAA3ElEQVQ4jc3TQUqDMRQE4M9aXLXduqmuCr1D1QPUexT1OFJKT6D0BCLu9ByKQi1ueoGCUBeJGtP8VQoFB0KSl3mTSfLCFnCCCWZYYIor9BJOp5RYxxhPGKCNPRziHC+4RBevJYExbtBIYstk3MR9dJXGv2w/Zsm5QCfuvCwJTKLtHCvEKsyEM28ssBAuLE0sWsUu3j8ntdjPsZ+QdpKWo423XOAB/T+6PY38HzgW3r/5S3JLqIdeaXGE2zUiLdxhWKVejyLPOMOB70q8iDsPI28tjnAt/IGqv/CP8AG5AS6gOPyB6QAAAABJRU5ErkJggg=="
$start_icon = Base64toIcon -base64 "iVBORw0KGgoAAAANSUhEUgAAABAAAAAQCAYAAAAf8/9hAAAAIGNIUk0AAHolAACAgwAA+f8AAIDoAABSCAABFVgAADqXAAAXb9daH5AAAAB3SURBVHjaxNOtDQJREEXhb6EBBA2gEBSBpBxqQVIHWUUR67YLGlhzUQ8BCHiPsDc5bnImmZ8uiZYstCbJA1yx+7Q+yYsgmHDGulZQuOGIZa2gMOLQIihcsHkW/HwL7zoP2P91iBNOWNWsscf2m0PqZv+F+QX3AQBWsuXyzXa1GAAAAABJRU5ErkJggg=="
$stop_icon = Base64toIcon -base64 "iVBORw0KGgoAAAANSUhEUgAAABAAAAAQCAYAAAAf8/9hAAAAIGNIUk0AAHolAACAgwAA+f8AAIDoAABSCAABFVgAADqXAAAXb9daH5AAAAA4SURBVHjaYvz//z8DJYCJgULAAmMwMjIS5ZT///8zUtUFowaMGsDAwMDAOOB5YeANAAAAAP//AwAD+wob1fih/QAAAABJRU5ErkJggg=="
$remove_icon = Base64toIcon -base64 "iVBORw0KGgoAAAANSUhEUgAAABAAAAAQCAYAAAAf8/9hAAAABmJLR0QA/wD/AP+gvaeTAAAAXUlEQVQ4je2TwQrAIAhAX/uEfrnYN+epLo2F4VrgbnsQCsrTg8FHnIAAtce8KxAg9jwCxWoMQ143BgSrsJJM9WPRqOOEJXjNL3AQaMYL1Dxe5EXm/gP6CZBc1nSlAetOFiRrGKtBAAAAAElFTkSuQmCC"
$check_status_icon = Base64toIcon -base64 "iVBORw0KGgoAAAANSUhEUgAAABAAAAAQCAYAAAAf8/9hAAAABmJLR0QA/wD/AP+gvaeTAAAAxUlEQVQ4jdXPv04CQRCA8Z9gLjYQWhNpaIml8QloCERj4kvQ8QgS4h98Hu0Ij6Gh5jSxlMZOC+cMxd1eK1+yyc7szDez/EcyPOANOe4jB4/YYlEUN0oEM/RxhnOc4ibevjDAJLVBjpOduBu5gndMi+Aw/Zs/vnfux3XFd3iKLbp4xm2FrJQsJDk20ZxVFTdLckNco4cjHOAD67rJYtoLLtCKc4lXzOuax9HcKXnrhGSUEqxichVXWKYEW78rV9HGZ0qwp/wAjhQf8o7RazcAAAAASUVORK5CYII="
$watch_stream_icon = Base64toIcon -base64 "iVBORw0KGgoAAAANSUhEUgAAABAAAAAQCAYAAAAf8/9hAAAABmJLR0QA/wD/AP+gvaeTAAAAZklEQVQ4jd3SwQmAQAwEwFHEAqxZfGihtuAJoi8/engeCoILISRsNksIL6BDwJoRAe0uENBkLm0w7cV6yCmc+HcHo0JlRPmug+zBPzt49AcVBoyoMwRm9ClSiSVFKiK9q1vE+B9jA3+AJxKa4t2TAAAAAElFTkSuQmCC"
$copy_name_icon = Base64toIcon -base64 "iVBORw0KGgoAAAANSUhEUgAAABAAAAAQCAYAAAAf8/9hAAAABmJLR0QA/wD/AP+gvaeTAAAAXElEQVQ4jd2SzQrAIAyDP/YIvrKyZ169uNvwsGj9gcEChUJIGmhgE07AgNIYA5IyMCB0jgTgUmRxJpVpRgxe04waPPvhFEr8yCDj64EpMuFrYqw08nPLrZxJ8yFuJ3UvowhAfiMAAAAASUVORK5CYII="
$up_icon = Base64toIcon -base64 "iVBORw0KGgoAAAANSUhEUgAAABAAAAAQCAYAAAAf8/9hAAAABmJLR0QA/wD/AP+gvaeTAAAAQElEQVQ4jWNgGAU0B6VQTLbmm1BMsiFFUI3SDAwM4gwMDFcZGBiqyNEMA0Qbgk0z0YYUMTAw/CcSE+2dUUAEAADLExbNk7QLxgAAAABJRU5ErkJggg=="
$down_icon = Base64toIcon -base64 "iVBORw0KGgoAAAANSUhEUgAAABAAAAAQCAYAAAAf8/9hAAAABmJLR0QA/wD/AP+gvaeTAAAAQ0lEQVQ4jWNgGAVUBUUMDAz/icRV+Ay5xcDAII1FToKBgeEaPs34DCFaMzZDSNYMA6UMDAw3obiUVM3IhpCteRQQCQCrixbW1FZU3wAAAABJRU5ErkJggg=="

#### Timers #######################################################################################################################################################################

$Timer_StatusCheck = New-Object System.Windows.Forms.Timer
$Timer_StatusCheck.Interval = [int]$status_check_interval * 60000
$Timer_StatusCheck.Add_Tick({
  CheckStatusAll
})

###################################################################################################################################################################################
#### Main Form ####################################################################################################################################################################
###################################################################################################################################################################################

$MainForm = New-Object System.Windows.Forms.Form
$MainForm.Text = "Streamlink GUI 2.04"
$MainForm.AutoSize = $true
$MainForm.AllowDrop = $true
$MainForm.Icon = $main_icon

if ($center_screen_position -eq 1) {
  $MainForm.StartPosition = "CenterScreen"
} else {
  $MainForm.StartPosition = "Manual"
  $MainForm.Location = "$location_x, $location_y"
}

$MainForm.Add_Activated({
  $MainForm.Opacity = 1
})

$MainForm.Add_Deactivate({
  $MainForm.Opacity = 0.9
})

$MainForm.Add_Click({
  $DataGridView.ClearSelection()
})

$MainForm.Add_FormClosing({
  if ($DataGridView.RowCount -ne 0 -and $autosave_model_list -ne 1) {
    $msgBoxInput = [System.Windows.Forms.MessageBox]::Show("Save current list?", "Question", "YesNo", "Question")
    if ($msgBoxInput -eq "Yes") {
      SaveModelList
      Get-Job | Stop-Job
      Get-Job | Remove-Job -Force
    }
  }
  if ($show_notify_icon -eq 1) {
    $NotifyIcon.Visible = $false
  }
})

$MainForm.Add_DragEnter({
  $_.Effect = [Windows.Forms.DragDropEffects]::Copy
})

$MainForm.Add_DragDrop({
  $TextBox_ModelName.Clear()
  $TextBox_ModelName.AppendText($_.Data.GetData("Text"))
  AddNewRow
})

$MainForm.Add_SizeChanged({
  if ($MainForm.WindowState -eq "Minimized") {
    if ($MenuItem_CheckStatusWhenMinimized.Checked -eq $true) {
      $Timer_StatusCheck.Start()
    }
  } else {
    if ($MenuItem_CheckStatusWhenMinimized.Checked -eq $true) {
      $Timer_StatusCheck.Stop()
    }

    if ($show_notify_icon -eq 1) {
      if ($NotifyIcon.Icon -eq $waiting_icon -or $NotifyIcon.Icon -eq $waiting_icon_green) {
        $NotifyIcon.Icon = $waiting_icon
      } else {
        $NotifyIcon.Icon = $main_icon
      }
    }

  }
})

#### Notify Icon ##################################################################################################################################################################

if ($show_notify_icon -eq 1) {
  $NotifyIcon = New-Object System.Windows.Forms.NotifyIcon
  $NotifyIcon.Text = "Streamlink GUI"
  $NotifyIcon.Icon = $main_icon
  $NotifyIcon.Visible = $true

#### Enable Notifications

  $NotifyIcon_MenuItem_EnableNotifications = New-Object System.Windows.Forms.ToolStripMenuItem
  $NotifyIcon_MenuItem_EnableNotifications.Text = "Enable Notifications"
  if ($enable_notifications -eq 1 -and $auto_status_check_when_minimized -eq 1) {
    $NotifyIcon_MenuItem_EnableNotifications.Checked = $true
  } elseif ($auto_status_check_when_minimized -ne 1) {
    $NotifyIcon_MenuItem_EnableNotifications.Checked = $false
    $NotifyIcon_MenuItem_EnableNotifications.Enabled = $false
  }
  $NotifyIcon_MenuItem_EnableNotifications.Add_Click({
    if ($NotifyIcon_MenuItem_EnableNotifications.Checked -eq $false) {
      $NotifyIcon_MenuItem_EnableNotifications.Checked = $true
    } else {
      $NotifyIcon_MenuItem_EnableNotifications.Checked = $false
    }
  })

#### Open Records Folder

  $NotifyIcon_MenuItem_OpenSaveDirectory = New-Object System.Windows.Forms.ToolStripMenuItem
  $NotifyIcon_MenuItem_OpenSaveDirectory.Text = "Open Records Folder"
  $NotifyIcon_MenuItem_OpenSaveDirectory.Image = $open_folder_icon
  $NotifyIcon_MenuItem_OpenSaveDirectory.Add_Click({
     OpenRecordsFolder
  })

#### Start All

  $NotifyIcon_MenuItem_StartAll = New-Object System.Windows.Forms.ToolStripMenuItem
  $NotifyIcon_MenuItem_StartAll.Text = "Start All"
  $NotifyIcon_MenuItem_StartAll.Add_Click({
    StartAll
  })

#### Stop All

  $NotifyIcon_MenuItem_StopAll = New-Object System.Windows.Forms.ToolStripMenuItem
  $NotifyIcon_MenuItem_StopAll.Text = "Stop All"
  $NotifyIcon_MenuItem_StopAll.Add_Click({
    StopAll
  })

#### Exit

  $NotifyIcon_MenuItem_Exit = New-Object System.Windows.Forms.ToolStripMenuItem
  $NotifyIcon_MenuItem_Exit.Text = "Exit"
  $NotifyIcon_MenuItem_Exit.Add_Click({
    StopAllAndExit
  })

  $NotifyIcon.ContextMenuStrip = New-Object System.Windows.Forms.ContextMenuStrip
  $NotifyIcon.ContextMenuStrip.Items.AddRange(@($NotifyIcon_MenuItem_EnableNotifications, $NotifyIcon_MenuItem_OpenSaveDirectory, $NotifyIcon_MenuItem_StartAll, $NotifyIcon_MenuItem_StopAll, $NotifyIcon_MenuItem_Exit))

  $NotifyIcon.Add_Click({
    if ($_.Button -eq "Left") {
      if ($MainForm.WindowState -eq "Normal") {
        $MainForm.WindowState = "Minimized"
      } elseif ($MainForm.WindowState -eq "Minimized") {
        $MainForm.WindowState = "Normal"
      }
    }
  })
}

#### Status Bar ###################################################################################################################################################################

$StatusBar = New-Object System.Windows.Forms.StatusBar
$StatusBar.Text = "Ready"
$MainForm.Controls.Add($statusBar)

#### Menu Strip ###################################################################################################################################################################

#### Open List

$MenuItem_Open = New-Object System.Windows.Forms.ToolStripMenuItem
$MenuItem_Open.Text = "Open List"
$MenuItem_Open.Add_Click({
  if ($DataGridView.RowCount -ne 0) {
    $msgBoxInput = [System.Windows.Forms.MessageBox]::Show("Save current list?", "Question", "YesNoCancel", "Question")
    switch  ($msgBoxInput) {
      "Yes" {SaveModelList | OpenModelList}
      "No" {OpenModelList}
      "Cancel" {$TextBox_ModelName.Focus()}
    }
  } else {
    OpenModelList
  }
})

#### Save List

$MenuItem_Save = New-Object System.Windows.Forms.ToolStripMenuItem
$MenuItem_Save.Text = "Save List"
$MenuItem_Save.Add_Click({
  SaveModelList
})

#### Open Records Folder

$MenuItem_OpenSaveDirectory = New-Object System.Windows.Forms.ToolStripMenuItem
$MenuItem_OpenSaveDirectory.Text = "Open Records Folder"
$MenuItem_OpenSaveDirectory.Image = $open_folder_icon
$MenuItem_OpenSaveDirectory.Add_Click({
  OpenRecordsFolder
})

#### Start After Add

$MenuItem_StartRecAfterAdd = New-Object System.Windows.Forms.ToolStripMenuItem
$MenuItem_StartRecAfterAdd.Text = "Start After Add"
if ($enable_recording -eq 0) {
  $MenuItem_StartRecAfterAdd.Enabled = $false
} elseif ($start_rec_after_add -eq 1) {
  $MenuItem_StartRecAfterAdd.Checked = $true
}
$MenuItem_StartRecAfterAdd.Add_Click({
  if ($MenuItem_StartRecAfterAdd.Checked -eq $false) {
    $MenuItem_StartRecAfterAdd.Checked = $true
  } else {
    $MenuItem_StartRecAfterAdd.Checked = $false
  }
})

#### Always On Top

$MenuItem_AlwaysOnTop = New-Object System.Windows.Forms.ToolStripMenuItem
$MenuItem_AlwaysOnTop.Text = "Always On Top"
if ($always_on_top -eq 1) {
  $MenuItem_AlwaysOnTop.Checked = $true
  $MainForm.TopMost = $true
}
$MenuItem_AlwaysOnTop.Add_Click({
  if ($MainForm.TopMost -eq $false) {
    $MainForm.TopMost = $true
    $MenuItem_AlwaysOnTop.Checked = $true
  } else {
    $MainForm.TopMost = $false
    $MenuItem_AlwaysOnTop.Checked = $false
  }
})

#### Test Connection

$MenuItem_TestConnection = New-Object System.Windows.Forms.ToolStripMenuItem
$MenuItem_TestConnection.Text = "Test Connection"
$MenuItem_TestConnection.Add_Click({
  TestConnection
})

$MenuItem_File_Separator1 = New-Object System.Windows.Forms.ToolStripSeparator

#### Start All

$MenuItem_StartAll = New-Object System.Windows.Forms.ToolStripMenuItem
$MenuItem_StartAll.Text = "Start All"
if ($enable_recording -eq 0) {
  $MenuItem_StartAll.Enabled = $false
}
$MenuItem_StartAll.Add_Click({
  StartAll
})

$MenuItem_File_Separator2 = New-Object System.Windows.Forms.ToolStripSeparator

#### Stop All

$MenuItem_StopAll = New-Object System.Windows.Forms.ToolStripMenuItem
$MenuItem_StopAll.Text = "Stop All"
$MenuItem_StopAll.Add_Click({
  StopAll
})

#### Stop All & Clear List

$MenuItem_StopAllAndClear = New-Object System.Windows.Forms.ToolStripMenuItem
$MenuItem_StopAllAndClear.Text = "Stop All && Clear List"
$MenuItem_StopAllAndClear.Add_Click({
  if ($DataGridView.RowCount -ne 0) {
    $msgBoxInput = [System.Windows.Forms.MessageBox]::Show("Clear current list?", "Question", "YesNo", "Question")
    if ($msgBoxInput -eq "Yes") {
      StopAllAndClear
    }
  }
})

#### Stop All & Exit

$MenuItem_Exit = New-Object System.Windows.Forms.ToolStripMenuItem
$MenuItem_Exit.Text = "Stop All && Exit"
$MenuItem_Exit.Add_Click({
  StopAllAndExit
})

#### Site #########################################################################################################################################################################

#### Chaturbate ###################################################################################################################################################################

#### CB Navigation 1

$MenuItem_CB_Navigation1 = New-Object System.Windows.Forms.ToolStripComboBox
$MenuItem_CB_Navigation1.DropDownStyle = "DropDownList"
ForEach ($item in "Featured", "Female", "Male", "Couple", "Trans") {
  $MenuItem_CB_Navigation1.Items.Add($item) | Out-Null
}
$MenuItem_CB_Navigation1.SelectedIndex = 0

#### CB Navigation 2

$MenuItem_CB_Navigation2 = New-Object System.Windows.Forms.ToolStripComboBox
$MenuItem_CB_Navigation2.DropDownStyle = "DropDownList"
ForEach ($item in "All", "Teen", "18 to 21", "20 to 30", "30 to 50", "Mature", "North American", "Other Region", "Euro Russian", "Asian", "South American", "Exhibitionist", "HD", "New") {
  $MenuItem_CB_Navigation2.Items.Add($item) | Out-Null
}
$MenuItem_CB_Navigation2.SelectedIndex = 0

#### CB Page

$MenuItem_CB_Page = New-Object System.Windows.Forms.ToolStripComboBox
$MenuItem_CB_Page.ToolTipText = "Page number"
ForEach ($page in "1", "2", "3", "4", "5", "6", "7", "8", "9", "10") {
  $MenuItem_CB_Page.Items.Add($page) | Out-Null
}
$MenuItem_CB_Page.SelectedIndex = 0

#### CB ThroughProxy

$MenuItem_CB_ThroughProxy = New-Object System.Windows.Forms.ToolStripMenuItem
$MenuItem_CB_ThroughProxy.Text = "Through Proxy"
$MenuItem_CB_ThroughProxy.CheckOnClick = $true
$MenuItem_CB_ThroughProxy.Add_Click({
  $MenuItem_Site.ShowDropDown()
  $MenuItem_Chaturbate.ShowDropDown()
})
if ($to_use_proxy_list -ne 1) {
  $MenuItem_CB_ThroughProxy.Checked = $false
  $MenuItem_CB_ThroughProxy.Enabled = $false
} else {
  $MenuItem_CB_ThroughProxy.Checked = $true
}

#### Hide Male

$MenuItem_CB_HideMale = New-Object System.Windows.Forms.ToolStripMenuItem
$MenuItem_CB_HideMale.Text = "Hide Male"
$MenuItem_CB_HideMale.Checked = $true
$MenuItem_CB_HideMale.CheckOnClick = $true
$MenuItem_CB_HideMale.Add_Click({
  $MenuItem_Site.ShowDropDown()
  $MenuItem_Chaturbate.ShowDropDown()
})

#### Hide Trans

$MenuItem_CB_HideTrans = New-Object System.Windows.Forms.ToolStripMenuItem
$MenuItem_CB_HideTrans.Text = "Hide Trans"
$MenuItem_CB_HideTrans.Checked = $true
$MenuItem_CB_HideTrans.CheckOnClick = $true
$MenuItem_CB_HideTrans.Add_Click({
  $MenuItem_Site.ShowDropDown()
  $MenuItem_Chaturbate.ShowDropDown()
})

#### Hide Colombia

$MenuItem_CB_HideColombia = New-Object System.Windows.Forms.ToolStripMenuItem
$MenuItem_CB_HideColombia.Text = "Hide Colombia"
$MenuItem_CB_HideColombia.Checked = $true
$MenuItem_CB_HideColombia.CheckOnClick = $true
$MenuItem_CB_HideColombia.Add_Click({
  $MenuItem_Site.ShowDropDown()
  $MenuItem_Chaturbate.ShowDropDown()
})

####

$MenuItem_CB_ThroughProxy.Add_CheckStateChanged({
  $MenuItem_CB_HideMale.Enabled = $MenuItem_CB_ThroughProxy.Checked
  $MenuItem_CB_HideTrans.Enabled = $MenuItem_CB_ThroughProxy.Checked
  $MenuItem_CB_HideColombia.Enabled = $MenuItem_CB_ThroughProxy.Checked
})
if ($to_use_proxy_list -ne 1) {
  $MenuItem_CB_HideMale.Enabled = $false
  $MenuItem_CB_HideTrans.Enabled = $false
  $MenuItem_CB_HideColombia.Enabled = $false
}

#### Separator 1

$MenuItem_CB_Separator1 = New-Object System.Windows.Forms.ToolStripSeparator

#### CB Open In Browser

$MenuItem_CB_OpenInBrowser = New-Object System.Windows.Forms.ToolStripMenuItem
$MenuItem_CB_OpenInBrowser.Text = "Open In Browser"
$MenuItem_CB_OpenInBrowser.Add_Click({
  MainFormDisabled
  OpenModelListInBrowserCB
  MainFormEnabled
})

#### Chaturbate

$MenuItem_Chaturbate = New-Object System.Windows.Forms.ToolStripMenuItem
$MenuItem_Chaturbate.Text = "Chaturbate"
$MenuItem_Chaturbate.DropDownItems.AddRange(@($MenuItem_CB_Navigation1, $MenuItem_CB_Navigation2, $MenuItem_CB_Page, $MenuItem_CB_ThroughProxy, $MenuItem_CB_HideMale, 
                                              $MenuItem_CB_HideTrans, $MenuItem_CB_HideColombia, $MenuItem_CB_Separator1, $MenuItem_CB_OpenInBrowser))

#### BongaCams ####################################################################################################################################################################

#### BC Navigation

$MenuItem_BC_Navigation = New-Object System.Windows.Forms.ToolStripComboBox
$MenuItem_BC_Navigation.DropDownStyle = "DropDownList"
ForEach ($item in "Females", "Couples", "Males", "Transsexuals", "New") {
  $MenuItem_BC_Navigation.Items.Add($item) | Out-Null
}
$MenuItem_BC_Navigation.SelectedIndex = 0

#### BC SortBy

$MenuItem_BC_SortBy = New-Object System.Windows.Forms.ToolStripComboBox
$MenuItem_BC_SortBy.ToolTipText = "Sort by"
$MenuItem_BC_SortBy.DropDownStyle = "DropDownList"
ForEach ($item in "Camscore", "Most Popular rooms", "Just logged on", "New Models", "Lovers") {
  $MenuItem_BC_SortBy.Items.Add($item) | Out-Null
}
$MenuItem_BC_SortBy.SelectedIndex = 1

#### BC Minimum Viewers

$MenuItem_BC_MinViewers = New-Object System.Windows.Forms.ToolStripTextBox
$MenuItem_BC_MinViewers.Text = 0
$MenuItem_BC_MinViewers.ToolTipText = "Minimum Viewers"

#### BC ThroughProxy

$MenuItem_BC_ThroughProxy = New-Object System.Windows.Forms.ToolStripMenuItem
$MenuItem_BC_ThroughProxy.Text = "Through Proxy"
$MenuItem_BC_ThroughProxy.Checked = $true
$MenuItem_BC_ThroughProxy.CheckOnClick = $true
$MenuItem_BC_ThroughProxy.Add_Click({
  $MenuItem_Site.ShowDropDown()
  $MenuItem_BongaCams.ShowDropDown()
})
if ($to_use_proxy_list -ne 1) {
  $MenuItem_BC_ThroughProxy.Checked = $false
  $MenuItem_BC_ThroughProxy.Enabled = $false
}

#### Separator 1

$MenuItem_BC_Separator1 = New-Object System.Windows.Forms.ToolStripSeparator

#### BC Open In Browser

$MenuItem_BC_OpenInBrowser = New-Object System.Windows.Forms.ToolStripMenuItem
$MenuItem_BC_OpenInBrowser.Text = "Open In Browser"
$MenuItem_BC_OpenInBrowser.Add_Click({
  MainFormDisabled
  OpenModelListInBrowserBC
  MainFormEnabled
})

#### BongaCams

$MenuItem_BongaCams = New-Object System.Windows.Forms.ToolStripMenuItem
$MenuItem_BongaCams.Text = "BongaCams"
$MenuItem_BongaCams.DropDownItems.AddRange(@($MenuItem_BC_Navigation, $MenuItem_BC_SortBy, $MenuItem_BC_MinViewers, $MenuItem_BC_ThroughProxy, $MenuItem_BC_Separator1, $MenuItem_BC_OpenInBrowser))

#### Model #######################################################################################################################################################################

#### Watch Streams In VLC

$MenuItem_WatchStreamsInVLC = New-Object System.Windows.Forms.ToolStripMenuItem
$MenuItem_WatchStreamsInVLC.Text = "Watch Streams In VLC"
if ($open_stream_in_vlc -eq 1 -and $vlc_not_found -ne 1) {
  $MenuItem_WatchStreamsInVLC.Checked = $true
} elseif ($vlc_not_found -eq 1) {
  $MenuItem_WatchStreamsInVLC.Checked = $false
  $MenuItem_WatchStreamsInVLC.Enabled = $false
}
$MenuItem_WatchStreamsInVLC.Add_Click({
  $MenuItem_WatchStreamsInVLC.Checked = $true
  $MenuItem_WatchStreamsInBrowser.Checked = $false
})

#### Watch Streams In Browser

$MenuItem_WatchStreamsInBrowser = New-Object System.Windows.Forms.ToolStripMenuItem
$MenuItem_WatchStreamsInBrowser.Text = "Watch Streams In Browser"
if ($open_stream_in_vlc -ne 1 -or $vlc_not_found -eq 1) {
  $MenuItem_WatchStreamsInBrowser.Checked = $true
}
$MenuItem_WatchStreamsInBrowser.Add_Click({
  $MenuItem_WatchStreamsInBrowser.Checked = $true
  $MenuItem_WatchStreamsInVLC.Checked = $false
})

#### Separator 1

$MenuItem_Model_Separator1 = New-Object System.Windows.Forms.ToolStripSeparator

#### Copy Name to Clipboard

$MenuItem_CopyNameToClipboard = New-Object System.Windows.Forms.ToolStripMenuItem
$MenuItem_CopyNameToClipboard.Text = "Copy Name to Clipboard"
$MenuItem_CopyNameToClipboard.Add_Click({
  CopyNameToClipboard
})

#### Copy M3U8 to Clipboard

$MenuItem_CopyM3U8ToClipboard = New-Object System.Windows.Forms.ToolStripMenuItem
$MenuItem_CopyM3U8ToClipboard.Text = "Copy .m3u8 to Clipboard"
$MenuItem_CopyM3U8ToClipboard.Add_Click({
  MainFormDisabled
  $m3u8 = $null
  $m3u8 = GetM3U8 -site $DataGridView.SelectedCells[2].FormattedValue -model_name $DataGridView.SelectedCells[1].FormattedValue
  if ($m3u8) {
    Set-Clipboard -Value $m3u8
  }
  MainFormEnabled
})

#### Show Available Streams

$MenuItem_ShowAvailableStreams = New-Object System.Windows.Forms.ToolStripMenuItem
$MenuItem_ShowAvailableStreams.Text = "Show Available Streams"
$MenuItem_ShowAvailableStreams.Add_Click({
  if ($DataGridView.SelectedRows.Count -eq 1) {
    MainFormDisabled
    ShowAvailableStreams -site $DataGridView.SelectedCells[2].FormattedValue -model_name $DataGridView.SelectedCells[1].FormattedValue
    MainFormEnabled
  }
})

#### Last Broadcast

$MenuItem_ShowLastBroadcastTime = New-Object System.Windows.Forms.ToolStripMenuItem
$MenuItem_ShowLastBroadcastTime.Text = "Last Broadcast"
$MenuItem_ShowLastBroadcastTime.Add_Click({
  if ($DataGridView.SelectedRows.Count -eq 1) {
    MainFormDisabled
    ShowLastBroadcastTime -site $DataGridView.SelectedCells[2].FormattedValue -model_name $DataGridView.SelectedCells[1].FormattedValue
    MainFormEnabled
  }
})

#### Change Quality

$MenuItem_ChangeQuality = New-Object System.Windows.Forms.ToolStripMenuItem
$MenuItem_ChangeQuality.Text = "Change Quality"
$MenuItem_ChangeQuality.Add_Click({
  ChangeQuality
})

#### Change Quality & Restart

$MenuItem_ChangeQualityAndRestart = New-Object System.Windows.Forms.ToolStripMenuItem
$MenuItem_ChangeQualityAndRestart.Text = "Change Quality && Restart"
$MenuItem_ChangeQualityAndRestart.Add_Click({
  ChangeQualityAndRestart
})

#### Status ######################################################################################################################################################################

#### Check Status

$MenuItem_CheckStatus = New-Object System.Windows.Forms.ToolStripMenuItem
$MenuItem_CheckStatus.Text = "Check Status"
$MenuItem_CheckStatus.Add_Click({
  CheckStatus
})

#### Check Status (All)

$MenuItem_CheckStatusAll = New-Object System.Windows.Forms.ToolStripMenuItem
$MenuItem_CheckStatusAll.Text = "Check Status (All)"
$MenuItem_CheckStatusAll.Add_Click({
  CheckStatusAll
})

#### Check Status (Stopped)

$MenuItem_CheckStatusStopped = New-Object System.Windows.Forms.ToolStripMenuItem
$MenuItem_CheckStatusStopped.Text = "Check Status (Stopped)"
$MenuItem_CheckStatusStopped.Add_Click({
  CheckStatusStopped
})

#### Use Proxy For MFC Status Check

$MenuItem_CheckMFCStatusThroughProxy = New-Object System.Windows.Forms.ToolStripMenuItem
$MenuItem_CheckMFCStatusThroughProxy.Text = "Use Proxy For MFC Status Check"
if ($to_use_proxy_list_for_mfc_status_check -eq 1 -and $to_use_proxy_list -eq 1) {
  $MenuItem_CheckMFCStatusThroughProxy.Checked = $true
}
$MenuItem_CheckMFCStatusThroughProxy.Add_Click({
  if ($MenuItem_CheckMFCStatusThroughProxy.Checked -eq $false) {
    $MenuItem_CheckMFCStatusThroughProxy.Checked = $true
  } else {
    $MenuItem_CheckMFCStatusThroughProxy.Checked = $false
  }
})

#### Separator 1

$MenuItem_Status_Separator1 = New-Object System.Windows.Forms.ToolStripSeparator

#### Check Status When Minimized

$MenuItem_CheckStatusWhenMinimized = New-Object System.Windows.Forms.ToolStripMenuItem
$MenuItem_CheckStatusWhenMinimized.Text = "Check Status When Minimized"
if ($auto_status_check_when_minimized -eq 1) {
  $MenuItem_CheckStatusWhenMinimized.Checked = $true
}
$MenuItem_CheckStatusWhenMinimized.Add_Click({
  if ($MenuItem_CheckStatusWhenMinimized.Checked -eq $false) {
    $MenuItem_CheckStatusWhenMinimized.Checked = $true
    $MenuItem_AutoStatusCheckInterval.Enabled = $true
    $MenuItem_StartRecordingIfOnline.Enabled = $true
    if ($show_notify_icon -eq 1) {
      $NotifyIcon_MenuItem_EnableNotifications.Enabled = $true
      if ($enable_notifications -eq 1) {
        $NotifyIcon_MenuItem_EnableNotifications.Checked = $true
      }
    }
  } else {
    $MenuItem_CheckStatusWhenMinimized.Checked = $false
    $MenuItem_AutoStatusCheckInterval.Enabled = $false
    $MenuItem_StartRecordingIfOnline.Enabled = $false
    if ($show_notify_icon -eq 1) {
      $NotifyIcon_MenuItem_EnableNotifications.Checked = $false
      $NotifyIcon_MenuItem_EnableNotifications.Enabled = $false
    }
  }
})

#### Auto Status Check Interval ComboBox

$MenuItem_AutoStatusCheckIntervalComboBox = New-Object System.Windows.Forms.ToolStripComboBox
$MenuItem_AutoStatusCheckIntervalComboBox.Text = $status_check_interval
ForEach ($interval in "3", "4", "5", "6", "7", "8", "9", "10", "15", "20", "25", "30") {
  $MenuItem_AutoStatusCheckIntervalComboBox.Items.Add($interval) | Out-Null
}
$MenuItem_AutoStatusCheckIntervalComboBox.Add_SelectedIndexChanged({
  $Timer_StatusCheck.Interval = [int]$MenuItem_AutoStatusCheckIntervalComboBox.Text * 60000
})
$MenuItem_AutoStatusCheckIntervalComboBox.Add_TextUpdate({
  $Timer_StatusCheck.Interval = [int]$MenuItem_AutoStatusCheckIntervalComboBox.Text * 60000
})

#### Status Check Interval (Minutes)

$MenuItem_AutoStatusCheckInterval = New-Object System.Windows.Forms.ToolStripMenuItem
$MenuItem_AutoStatusCheckInterval.Text = "Status Check Interval (Minutes)"
$MenuItem_AutoStatusCheckInterval.Image = $timer_icon
$MenuItem_AutoStatusCheckInterval.DropDownItems.Add($MenuItem_AutoStatusCheckIntervalComboBox) | Out-Null

#### Start Recording If Online

$MenuItem_StartRecordingIfOnline = New-Object System.Windows.Forms.ToolStripMenuItem
$MenuItem_StartRecordingIfOnline.Text = 'Start Recording If Status "Online"'
$MenuItem_StartRecordingIfOnline.CheckOnClick = $true
if ($start_rec_if_online -eq 1) {
  $MenuItem_StartRecordingIfOnline.Checked = $true
}

#### Search ######################################################################################################################################################################

#### Cshive

$MenuItem_Cshive = New-Object System.Windows.Forms.ToolStripMenuItem
$MenuItem_Cshive.Text = "Cshive"
$MenuItem_Cshive.Add_Click({
  if ($DataGridView.SelectedRows.Count -ge 1) {
    MainFormDisabled
    ForEach ($row in $DataGridView.SelectedRows) {
      $html_page_name = $row.Cells[1].Value + "_search.htm"
      "<form action=`"https://cshive.com/search`" method=`"post`" name=`"searchform`"><input type=`"hidden`" name=`"search`" value=`"$($row.Cells[1].Value)`"></form><script>document.searchform.submit()</script>" > $html_page_name; start $html_page_name
    }
    Sleep 3
    ForEach ($row in $DataGridView.SelectedRows) {
      $html_page_name = $row.Cells[1].Value + "_search.htm"
      Remove-Item $html_page_name
    }
    MainFormEnabled
  }
})

#### ForumSmotri

$MenuItem_ForumSmotri = New-Object System.Windows.Forms.ToolStripMenuItem
$MenuItem_ForumSmotri.Text = "ForumSmotri"
$MenuItem_ForumSmotri.Add_Click({
  if ($DataGridView.SelectedRows.Count -eq 1) {
    start "https://forumsmotri.club/index.php?do=search&subaction=search&story=$($DataGridView.SelectedCells[1].Value)&titleonly=1"
  }
})

#### Rec-Tube

$MenuItem_RecTube = New-Object System.Windows.Forms.ToolStripMenuItem
$MenuItem_RecTube.Text = "Rec-Tube"
$MenuItem_RecTube.Add_Click({
  if ($DataGridView.SelectedRows.Count -eq 1) {
    start "https://rec-tube.com/search/$($DataGridView.SelectedCells[1].Value)/"
  }
})

#### Recurbate

$MenuItem_Recurbate = New-Object System.Windows.Forms.ToolStripMenuItem
$MenuItem_Recurbate.Text = "Recurbate"
$MenuItem_Recurbate.Add_Click({
  if ($DataGridView.SelectedRows.Count -eq 1) {
    start "https://recurbate.com/?performer=$($DataGridView.SelectedCells[1].Value)"
  }
})

#### CamWhores

$MenuItem_CamWhores = New-Object System.Windows.Forms.ToolStripMenuItem
$MenuItem_CamWhores.Text = "CamWhores"
$MenuItem_CamWhores.Add_Click({
  if ($DataGridView.SelectedRows.Count -eq 1) {
    start "http://www.camwhores.tv/search/$($DataGridView.SelectedCells[1].Value)/"
  }
})

#### NobodyHome

$MenuItem_NobodyHome = New-Object System.Windows.Forms.ToolStripMenuItem
$MenuItem_NobodyHome.Text = "NobodyHome"
$MenuItem_NobodyHome.Add_Click({
  if ($DataGridView.SelectedRows.Count -eq 1) {
    start "https://nobodyhome.tv/search.php?keywords=$($DataGridView.SelectedCells[1].Value)&action=do_search&postthread=1"
  }
})

#### MyWebGirls

$MenuItem_MyWebGirls = New-Object System.Windows.Forms.ToolStripMenuItem
$MenuItem_MyWebGirls.Text = "MyWebGirls"
$MenuItem_MyWebGirls.Add_Click({
  if ($DataGridView.SelectedRows.Count -eq 1) {
    start "http://www.mywebgirls.tv/search/?q=$($DataGridView.SelectedCells[1].Value)"
  }
})

#### CamVault

$MenuItem_CamVault = New-Object System.Windows.Forms.ToolStripMenuItem
$MenuItem_CamVault.Text = "CamVault"
$MenuItem_CamVault.Add_Click({
  if ($DataGridView.SelectedRows.Count -eq 1) {
    start "https://camvault.xyz/gallery/index/search/$($DataGridView.SelectedCells[1].Value)"
  }
})

####

$MenuItem_File = New-Object System.Windows.Forms.ToolStripMenuItem
$MenuItem_File.DropDownItems.AddRange(@($MenuItem_Open, $MenuItem_Save, $MenuItem_OpenSaveDirectory, $MenuItem_StartRecAfterAdd, $MenuItem_AlwaysOnTop, $MenuItem_TestConnection, 
                                        $MenuItem_File_Separator1, $MenuItem_StartAll, $MenuItem_File_Separator2, $MenuItem_StopAll, $MenuItem_StopAllAndClear, $MenuItem_Exit))
$MenuItem_File.Text = "File"

$MenuItem_Model = New-Object System.Windows.Forms.ToolStripMenuItem
$MenuItem_Model.DropDownItems.AddRange(@($MenuItem_WatchStreamsInVLC, $MenuItem_WatchStreamsInBrowser, $MenuItem_Model_Separator1, $MenuItem_CopyNameToClipboard, 
                                         $MenuItem_CopyM3U8ToClipboard, $MenuItem_ShowAvailableStreams, $MenuItem_ShowLastBroadcastTime, $MenuItem_ChangeQuality, 
                                         $MenuItem_ChangeQualityAndRestart))
$MenuItem_Model.Text = "Model"

$MenuItem_Status = New-Object System.Windows.Forms.ToolStripMenuItem
$MenuItem_Status.DropDownItems.AddRange(@($MenuItem_CheckStatus, $MenuItem_CheckStatusAll, $MenuItem_CheckStatusStopped, $MenuItem_CheckMFCStatusThroughProxy, 
                                          $MenuItem_Status_Separator1, $MenuItem_CheckStatusWhenMinimized, $MenuItem_AutoStatusCheckInterval, $MenuItem_StartRecordingIfOnline))
$MenuItem_Status.Text = "Status"

$MenuItem_Site = New-Object System.Windows.Forms.ToolStripMenuItem
$MenuItem_Site.DropDownItems.AddRange(@($MenuItem_Chaturbate, $MenuItem_BongaCams))
$MenuItem_Site.Text = "Site"

$MenuItem_Search = New-Object System.Windows.Forms.ToolStripMenuItem
$MenuItem_Search.DropDownItems.AddRange(@($MenuItem_Cshive, $MenuItem_ForumSmotri, $MenuItem_RecTube, $MenuItem_Recurbate, $MenuItem_CamWhores, $MenuItem_NobodyHome, 
                                          $MenuItem_MyWebGirls, $MenuItem_CamVault))
$MenuItem_Search.Text = "Search"

####

$MenuStrip = New-Object System.Windows.Forms.MenuStrip
$MenuStrip.Items.AddRange(@($MenuItem_File, $MenuItem_Site, $MenuItem_Model, $MenuItem_Status, $MenuItem_Search))
$MenuStrip.Location = "0, 0"
$MenuStrip.ShowItemToolTips = $true
$MainForm.Controls.Add($MenuStrip)

#### Text Boxes ###################################################################################################################################################################

$TextBox_ModelName = New-Object System.Windows.Forms.TextBox
$TextBox_ModelName.Location = "10, 35"
$TextBox_ModelName.Width = 220
$MainForm.Controls.Add($TextBox_ModelName)

$TextBox_ModelName.Add_GotFocus({
  $DataGridView.ClearSelection()
})

$TextBox_ModelName.Add_KeyDown({
  if ($_.KeyCode -eq "Enter") {
    AddNewRow
    $_.SuppressKeyPress = $true
  }
})

$TextBox_ModelName.Add_DoubleClick({
  $TextBox_ModelName.Text = Get-Clipboard
  AddNewRow
})

$TextBox_OtherQuality = New-Object System.Windows.Forms.TextBox
$TextBox_OtherQuality.Location = "170,65"
$TextBox_OtherQuality.Width = 155
$TextBox_OtherQuality.Enabled = $false
$MainForm.Controls.Add($TextBox_OtherQuality)

#### Combo Boxes ##################################################################################################################################################################

$ComboBox_SiteList = New-Object System.Windows.Forms.ComboBox
$ComboBox_SiteList.Location = "240, 35"
$ComboBox_SiteList.Width = 85
$ComboBox_SiteList.DropDownStyle = "DropDownList"
$ComboBox_SiteList.Items.Add("Chaturbate") | Out-Null
$ComboBox_SiteList.Items.Add("BongaCams") | Out-Null
$ComboBox_SiteList.Items.Add("StripChat") | Out-Null
$ComboBox_SiteList.Items.Add("MyFreeCams") | Out-Null
$ComboBox_SiteList.SelectedIndex = 0
$MainForm.Controls.Add($ComboBox_SiteList)

$ComboBox_SiteList.Add_SelectedValueChanged({
  if ($ComboBox_SiteList.Text -eq "Chaturbate") {
    $ComboBox_VideoQuality.DataSource = "best", "2160p", "1440p", "1080p", "1080p,720p_alt,720p,best", "720p", "720p_alt", "720p_alt,720p,1080p,best", "480p", "240p", "other"
  }
  if ($ComboBox_SiteList.Text -eq "BongaCams") {
    $ComboBox_VideoQuality.DataSource = "best", "1080p", "720p", "480p", "240p", "other"
  }
  if ($ComboBox_SiteList.Text -eq "StripChat") {
    $ComboBox_VideoQuality.DataSource = "best", "720p", "480p", "240p"
    $TextBox_OtherQuality.Clear()
  }
  if ($ComboBox_SiteList.Text -eq "MyFreeCams") {
     $ComboBox_VideoQuality.DataSource = @("best")
  }
})

$ComboBox_VideoQuality = New-Object System.Windows.Forms.ComboBox
$ComboBox_VideoQuality.Location = "10, 65"
$ComboBox_VideoQuality.Width = 150
$ComboBox_VideoQuality.DropDownStyle = "DropDownList"
ForEach ($quality in "best", "2160p", "1440p", "1080p", "1080p,720p_alt,720p,best", "720p", "720p_alt", "720p_alt,720p,1080p,best", "480p", "240p", "other") {
  $ComboBox_VideoQuality.Items.Add($quality) | Out-Null
}
$ComboBox_VideoQuality.SelectedIndex = 0
$MainForm.Controls.Add($ComboBox_VideoQuality)

$ComboBox_VideoQuality.Add_SelectedValueChanged({
  if ($ComboBox_VideoQuality.Text -eq "other") {
    $TextBox_OtherQuality.Enabled = $true
  } else {
    $TextBox_OtherQuality.Clear()
    $TextBox_OtherQuality.Enabled = $false
  }
})

#### Buttons ######################################################################################################################################################################

#### Add

$Button_Add = New-Object System.Windows.Forms.Button
$Button_Add.Location = "335, 35"
$Button_Add.Width = 65
$Button_Add.Height = 50
$Button_Add.Text = "Add"
$MainForm.Controls.Add($Button_Add)

$Button_Add.Add_Click({
  AddNewRow
})

#### DataGridView #################################################################################################################################################################

$DataGridView = New-Object System.Windows.Forms.DataGridView
$DataGridView.Location = "10, 101"
$DataGridView.MaximumSize = "0, $([System.Windows.Forms.Screen]::PrimaryScreen.WorkingArea.Height - 250)"
$DataGridView.AutoSize = $true
$DataGridView.BackgroundColor = $MainForm.BackColor
$DataGridView.BorderStyle = "None"
$DataGridView.RowHeadersVisible = $false
$DataGridView.SelectionMode = "FullRowSelect"
$DataGridView.ColumnHeadersHeightSizeMode = "DisableResizing"
$DataGridView.AllowUserToDeleteRows = $false
$DataGridView.AllowUserToAddRows = $false
$DataGridView.AllowUserToResizeRows = $false
$DataGridView.ReadOnly = $true
$DataGridView.RowCount = 0
$DataGridView.ColumnCount = 5
$DataGridView.Columns[0].Width = 35
$DataGridView.Columns[1].Width = 135
$DataGridView.Columns[2].Width = 70
$DataGridView.Columns[3].Width = 55
$DataGridView.Columns[4].Width = 85
$DataGridView.Columns[0].Name = "No"
$DataGridView.Columns[1].Name = "Name"
$DataGridView.Columns[2].Name = "Site"
$DataGridView.Columns[3].Name = "Quality"
$DataGridView.Columns[4].Name = "Status"

$DataGridView_RecordColumn = New-Object System.Windows.Forms.DataGridViewImageColumn
$DataGridView_RecordColumn.Width = 14
$DataGridView_RecordColumn.DefaultCellStyle.Alignment = "MiddleLeft"
$DataGridView_RecordColumn.DefaultCellStyle.NullValue = $null
$DataGridView.Columns.Add($DataGridView_RecordColumn) | Out-Null

$MainForm.Controls.Add($DataGridView)

$DataGridView.Add_CellDoubleClick({
  OpenModelRoomInBrowser -site $DataGridView.SelectedCells[2].FormattedValue -model_name $DataGridView.SelectedCells[1].FormattedValue
})

$DataGridView.Add_KeyDown({
  if ($_.KeyCode -eq "Delete") {
    RemoveSelected
  }

  if ($_.KeyCode -eq "Q") {
    MoveRow -1
  }

  if ($_.KeyCode -eq "A") {
    MoveRow 1
  }
})

#### Context Menu #################################################################################################################################################################

$ContextMenuStrip = New-Object System.Windows.Forms.ContextMenuStrip

#### Start

$ToolStripMenuItem_Start = New-Object System.Windows.Forms.ToolStripMenuItem
$ToolStripMenuItem_Start.Text = "Start Rec."
$ToolStripMenuItem_Start.Image = $start_icon
$ContextMenuStrip.Items.Add($ToolStripMenuItem_Start) | Out-Null

if ($enable_recording -eq 0) {
  $ToolStripMenuItem_Start.Enabled = $false
}

$ToolStripMenuItem_Start.Add_Click({
  StartSelected
})

#### Stop

$ToolStripMenuItem_Stop = New-Object System.Windows.Forms.ToolStripMenuItem
$ToolStripMenuItem_Stop.Text = "Stop Rec."
$ToolStripMenuItem_Stop.Image = $stop_icon
$ContextMenuStrip.Items.Add($ToolStripMenuItem_Stop) | Out-Null

if ($enable_recording -eq 0) {
  $ToolStripMenuItem_Stop.Enabled = $false
}

$ToolStripMenuItem_Stop.Add_Click({
  StopSelected
})

#### Remove

$ToolStripMenuItem_Remove = New-Object System.Windows.Forms.ToolStripMenuItem
$ToolStripMenuItem_Remove.Text = "Remove"
$ToolStripMenuItem_Remove.Image = $remove_icon
$ContextMenuStrip.Items.Add($ToolStripMenuItem_Remove) | Out-Null

$ToolStripMenuItem_Remove.Add_Click({
  RemoveSelected
})

#### Separator 1

$ToolStripMenuItem_Separator_1 = New-Object System.Windows.Forms.ToolStripSeparator
$ContextMenuStrip.Items.Add($ToolStripMenuItem_Separator_1) | Out-Null

#### Check Status

$ToolStripMenuItem_CheckStatus = New-Object System.Windows.Forms.ToolStripMenuItem
$ToolStripMenuItem_CheckStatus.Text = "Check Status"
$ToolStripMenuItem_CheckStatus.Image = $check_status_icon
$ContextMenuStrip.Items.Add($ToolStripMenuItem_CheckStatus) | Out-Null

$ToolStripMenuItem_CheckStatus.Add_Click({
  CheckStatus
})

#### Watch Stream

$ToolStripMenuItem_WatchStream = New-Object System.Windows.Forms.ToolStripMenuItem
$ToolStripMenuItem_WatchStream.Text = "Watch Stream"
$ToolStripMenuItem_WatchStream.Image = $watch_stream_icon
$ContextMenuStrip.Items.Add($ToolStripMenuItem_WatchStream) | Out-Null

$ToolStripMenuItem_WatchStream.Add_Click({
  WatchStream
})

#### Copy Name

$ToolStripMenuItem_CopyName = New-Object System.Windows.Forms.ToolStripMenuItem
$ToolStripMenuItem_CopyName.Text = "Copy Name"
$ToolStripMenuItem_CopyName.Image = $copy_name_icon
$ContextMenuStrip.Items.Add($ToolStripMenuItem_CopyName) | Out-Null

$ToolStripMenuItem_CopyName.Add_Click({
  CopyNameToClipboard
})

#### Separator 2

$ToolStripMenuItem_Separator_2 = New-Object System.Windows.Forms.ToolStripSeparator
$ContextMenuStrip.Items.Add($ToolStripMenuItem_Separator_2) | Out-Null

#### Move Up

$ToolStripMenuItem_MoveUp = New-Object System.Windows.Forms.ToolStripMenuItem
$ToolStripMenuItem_MoveUp.Text = "Move Up (Q)"
$ToolStripMenuItem_MoveUp.Image = $up_icon
$ContextMenuStrip.Items.Add($ToolStripMenuItem_MoveUp) | Out-Null

$ToolStripMenuItem_MoveUp.Add_Click({
  MoveRow -1
})

# Move Down

$ToolStripMenuItem_MoveDown = New-Object System.Windows.Forms.ToolStripMenuItem
$ToolStripMenuItem_MoveDown.Text = "Move Down (A)"
$ToolStripMenuItem_MoveDown.Image = $down_icon
$ContextMenuStrip.Items.Add($ToolStripMenuItem_MoveDown) | Out-Null

$ToolStripMenuItem_MoveDown.Add_Click({
  MoveRow 1
})

####

$DataGridView.Add_MouseDown({
  if ($_.Button -eq  [System.Windows.Forms.MouseButtons]::Right) {
    [System.Windows.Forms.DataGridView+HitTestInfo]$hit = $DataGridView.HitTest($_.X, $_.Y)
    if ($hit.Type -eq [System.Windows.Forms.DataGridViewHitTestType]::Cell) {
      if ($DataGridView.SelectedRows.Count -eq 0) {
        $DataGridView.Rows[$hit.RowIndex].Selected = $true
      } elseif ($DataGridView.SelectedRows.Count -eq 1) {
        $DataGridView.ClearSelection()
        $DataGridView.Rows[$hit.RowIndex].Selected = $true
      }
      $ContextMenuStrip.Show($DataGridView, $_.X, $_.Y)
    }
  }
})

#### ToolTips #####################################################################################################################################################################

$ToolTip = New-Object System.Windows.Forms.ToolTip
$ToolTip.SetToolTip($Button_Add, "Add model to list")
$ToolTip.SetToolTip($TextBox_OtherQuality, "Fallback streams can be specified by using`na comma-separated list: 720p,480p,best")
$ToolTip.SetToolTip($TextBox_ModelName, "Enter model name or link, double-click`nto add from clipboard")

#### Load Model List ##############################################################################################################################################################

if ($load_model_list_on_startup -eq "1") {
  if (Test-Path $model_list_path) {
    $model_list = @(Get-Content $model_list_path)
  }
  ForEach ($string in $model_list) {
    $n, $s, $q = $string.Split(";")
    DisplayRowInDataGrid -model_name $n -site $s -quality $q
  }
  if ($start_rec_after_load_list -eq 1 -and $enable_recording -eq 1) {
    StartAll
  }
  UpdateStatusBarText
}

#### Enable DataGridView DoubleBuffered ###########################################################################################################################################

function Enable-DataGridViewDoubleBuffer {
  param ([Parameter(Mandatory = $true)]
  [System.Windows.Forms.DataGridView]$grid,
  [switch]$Disable)
  $type = $grid.GetType();
  $propInfo = $type.GetProperty("DoubleBuffered", ('Instance','NonPublic'))
  $propInfo.SetValue($grid, $Disable -eq $false, $null)
}

Enable-DataGridViewDoubleBuffer -Grid $DataGridView

$MainForm.ShowDialog()
