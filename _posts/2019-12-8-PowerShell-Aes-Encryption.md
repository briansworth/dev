---
layout: post
title: PowerShell - Aes Encryption
description: Encrypt with Aes in PowerShell.
---

### Overview
----

This post will go over a way to implement AES encryption with PowerShell.
It will allow you to protect a string with a password.


#### Encryption Example
----

Create a password as a secure string, and encrypt it using it.

```powershell
$password = Read-Host -AsSecureString
$encrypted = Protect-AesString -String 'Encrypt me!_!' -Password $password

Write-Output $encrypted.CipherText
```

#### Decryption Example
----

Decrypt the text from above using the same password.

```powershell
$encrypted | Unprotect-AesString -Password $password
```


### The Code

----

There is a fair bit of code needed to pull this off. 
The source code is available on 
[Github](https://github.com/briansworth/AesEncryption/) and below.
Cryptography is complicated, and I have done my best to implement best practices.

```powershell
Function DecryptSecureString
{
  Param(
    [SecureString]$SecureString
  )
  $bstr = [Runtime.InteropServices.Marshal]::SecureStringToBSTR($SecureString)
  $plain = [Runtime.InteropServices.Marshal]::PtrToStringAuto($bstr)
  return $plain
}

Function NewPasswordKey 
{
  [CmdletBinding()]
  Param(
    [SecureString]$Password,

    [String]$Salt
  )
  $saltBytes = [Text.Encoding]::ASCII.GetBytes($Salt) 
  $iterations = 1000
  $keySize = 256

  $clearPass = DecryptSecureString -SecureString $Password
  $passwordType = 'Security.Cryptography.Rfc2898DeriveBytes'
  $passwordDerive = New-Object -TypeName $passwordType `
    -ArgumentList @( 
      $Password, 
      $saltBytes, 
      $iterations
    )

  $keyBytes = $passwordDerive.GetBytes($keySize / 8)
  return $keyBytes
}

Class CipherInfo
{
  [String]$CipherText
  [Byte[]]$IV
  [String]$Salt

  CipherInfo([String]$CipherText, [Byte[]]$IV, [String]$Salt)
  {
    $this.CipherText = $CipherText
    $this.IV = $IV
    $this.Salt = $Salt
  }
}

Function Protect-AesString 
{
  [CmdletBinding()]
  Param(
    [Parameter(Position=0, Mandatory=$true, ValueFromPipeline=$true)]
    [String]$String,

    [Parameter(Position=1, Mandatory=$true)]
    [SecureString]$Password,

    [Parameter(Position=2)]
    [String]$Salt = 'qtsbp6j643ah8e0omygzwlv9u75xcfrk4j63fdane78w1zgxhucsytkirol0v25q',

    [Parameter(Position=3)]
    [Security.Cryptography.PaddingMode]$Padding = 'PKCS7'
  )
  Try 
  {
    $valueBytes = [Text.Encoding]::UTF8.GetBytes($String)
    [byte[]]$keyBytes = NewPasswordKey -Password $Password -Salt $Salt

    $cipher = [Security.Cryptography.SymmetricAlgorithm]::Create('AesManaged')
    $cipher.Mode = [Security.Cryptography.CipherMode]::CBC
    $cipher.Padding = $Padding
    $vectorBytes = $cipher.IV

    $encryptor = $cipher.CreateEncryptor($keyBytes, $vectorBytes)
    $stream = New-Object -TypeName IO.MemoryStream
    $writer = New-Object -TypeName Security.Cryptography.CryptoStream `
      -ArgumentList @(
        $stream,
        $encryptor,
        [Security.Cryptography.CryptoStreamMode]::Write
      )

    $writer.Write($valueBytes, 0, $valueBytes.Length)
    $writer.FlushFinalBlock()
    $encrypted = $stream.ToArray()

    $cipher.Clear()
    $encryptedValue = [Convert]::ToBase64String($encrypted)
    New-Object -TypeName CipherInfo `
      -ArgumentList @($encryptedValue, $vectorBytes, $Salt)
  }
  Catch
  {
    Write-Error $_
  }
}

Function Unprotect-AesString 
{
  [CmdletBinding(DefaultParameterSetName='String')]
  Param(
    [Parameter(Position=0, Mandatory=$true, ParameterSetName='String')]
    [Alias('EncryptedString')]
    [String]$String,

    [Parameter(Position=1, Mandatory=$true)]
    [SecureString]$Password,

    [Parameter(Position=2, ParameterSetName='String')]
    [String]$Salt = 'qtsbp6j643ah8e0omygzwlv9u75xcfrk4j63fdane78w1zgxhucsytkirol0v25q',

    [Parameter(Position=3, Mandatory=$true, ParameterSetName='String')]
    [Alias('Vector')]
    [Byte[]]$InitializationVector,

    [Parameter(Position=0, Mandatory=$true, ParameterSetName='CipherInfo', ValueFromPipeline=$true)]
    [CipherInfo]$CipherInfo,

    [Parameter(Position=3, ParameterSetName='String')]
    [Parameter(Position=2, ParameterSetName='CipherInfo')]
    [Security.Cryptography.PaddingMode]$Padding = 'PKCS7'
  )
  Process
  {
    Try
    {
      if ($PSCmdlet.ParameterSetName -eq 'CipherInfo')
      {
        $Salt = $CipherInfo.Salt
        $InitializationVector = $CipherInfo.IV
        $String = $CipherInfo.CipherText
      }
      $iv = $InitializationVector

      $valueBytes = [Convert]::FromBase64String($String)
      $keyBytes = NewPasswordKey -Password $Password -Salt $Salt

      $cipher = [Security.Cryptography.SymmetricAlgorithm]::Create('AesManaged')
      $cipher.Mode = [Security.Cryptography.CipherMode]::CBC
      $cipher.Padding = $Padding

      $decryptor = $cipher.CreateDecryptor($keyBytes, $iv)
      $stream = New-Object -TypeName IO.MemoryStream `
        -ArgumentList @(, $valueBytes)
      $reader = New-Object -TypeName Security.Cryptography.CryptoStream `
        -ArgumentList @(
          $stream,
          $decryptor,
          [Security.Cryptography.CryptoStreamMode]::Read
        )

      $decrypted = New-Object -TypeName Byte[] -ArgumentList $valueBytes.Length
      $decryptedByteCount = $reader.Read($decrypted, 0, $decrypted.Length)
      $decryptedValue = [Text.Encoding]::UTF8.GetString(
        $decrypted,
        0,
        $decryptedByteCount
      )
      $cipher.Clear()
      return $decryptedValue
    }
    Catch
    {
      Write-Error $_
    }
  }
}
```


#### Example:
----

```powershell
$pass = Read-Host -AsSecureString
$info = Protect-AesString -String "Keep this super safe!!" -Password $pass

# Decrypt
Unprotect-AesString -CipherInfo $info -Password $pass
```
