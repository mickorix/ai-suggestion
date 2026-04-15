當然可以！這套方案結合了 OpenSSL 指令（後端簽名） 與 iOS Objective-C 代碼（前端驗證），能幫你建立一套完整的自動化憑證安全更新機制。
## 第一部分：後端環境 (OpenSSL 簽名指令)
首先，在你的開發電腦上生成一對專用的 RSA 金鑰對（這與 SSL 憑證無關，僅用於簽名清單）。

   1. 生成私鑰 (Private Key) - 請妥善保管，不可外洩：
   
   openssl genrsa -out pinning_private.pem 2048
   
   2. 提取公鑰 (Public Key) - 稍後要放入 iOS 代碼中：
   
   openssl rsa -in pinning_private.pem -pubout -out pinning_public.pem
   
   3. 對清單檔案 (pinning_list.json) 進行簽名：
   假設你的清單內容是 {"hashes":["hash1...","hash2..."]}，執行以下指令生成簽名檔：
   
   openssl dgst -sha256 -sign pinning_private.pem -out signature.bin pinning_list.json
   
   
------------------------------
## 第二部分：iOS 客戶端 (Objective-C 驗證代碼)
你需要將 pinning_public.pem 的內容放入 App，並在下載清單後進行驗證。
## 1. 導入公鑰與驗證工具
建議將公鑰內容以字串形式存放在代碼中。
```
#import <Security/Security.h>
// 這是從 pinning_public.pem 取得的內容 (去除 Header/Footer 和換行)static NSString *const kUpdatePublicKeyBase64 = @"MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA..."; 

- (BOOL)verifyData:(NSData *)data withSignature:(NSData *)signature {
    // 1. 將 Base64 公鑰轉為 SecKeyRef
    NSData *keyData = [[NSData alloc] initWithBase64EncodedString:kUpdatePublicKeyBase64 options:0];
    NSDictionary* options = @{
        (__bridge id)kSecAttrKeyType: (__bridge id)kSecAttrKeyTypeRSA,
        (__bridge id)kSecAttrKeyClass: (__bridge id)kSecAttrKeyClassPublic,
        (__bridge id)kSecAttrKeySizeInBits: @2048,
    };
    
    CFErrorRef error = NULL;
    SecKeyRef publicKey = SecKeyCreateWithData((__bridge CFDataRef)keyData, (__bridge CFDictionaryRef)options, &error);
    
    if (!publicKey) {
        NSLog(@"Public key creation failed: %@", error);
        return NO;
    }

    // 2. 驗證簽名 (iOS 10+)
    BOOL isInside = SecKeyVerifySignature(publicKey, 
                                           kSecKeyAlgorithmRSASignatureMessagePKCS1v15SHA256, 
                                           (__bridge CFDataRef)data, 
                                           (__bridge CFDataRef)signature, 
                                           &error);
    if (error) {
        NSLog(@"Verification error: %@", error);
    }
    
    CFRelease(publicKey);
    return isInside;
}
```
## 2. 整合進你的 didReceiveChallenge
驗證通過後，將 Hash 存入 NSUserDefaults。
```
- (void)URLSession:(NSURLSession *)session didReceiveChallenge:(NSURLAuthenticationChallenge *)challenge completionHandler:(void (^)(NSURLSessionAuthChallengeDisposition, NSURLCredential * _Nullable))completionHandler {
    
    // ... 取得當前 serverHashKey 的邏輯 (同你之前的代碼) ...

    // 取得動態清單
    NSArray *dynamicHashes = [[NSUserDefaults standardUserDefaults] objectForKey:@"RemotePinnedHashes"];
    
    // 驗證：只要匹配「寫死的初始 Hash」或「動態更新的 Hash」其中之一即可
    if ([serverHashKey isEqual:publicKeyHash] || [dynamicHashes containsObject:serverHashKey]) {
        NSLog(@"Pinning Success!");
        completionHandler(NSURLSessionAuthChallengeUseCredential, [NSURLCredential credentialForTrust:challenge.protectionSpace.serverTrust]);
    } else {
        NSLog(@"Pinning Failed - Key mismatch");
        completionHandler(NSURLSessionAuthChallengeCancelAuthenticationChallenge, nil);
    }
}
```
## 這套方案的優勢

   1. 安全性：即便 pinning_list.json 被放在公開的 GitHub Pages 上，因為有 signature.bin 的保護，駭客無法修改內容。
   2. 合適性 (Fitful)：當 Netlify 每 90 天換一次 Let's Encrypt 密鑰時，你只需要運行一次 OpenSSL 指令更新 GitHub 上的 JSON 和簽名，所有 App 就會自動信任新密鑰。

最後一個關鍵細節：你需要我提供如何將 .pem 公鑰轉換為 iOS 可讀 Base64 字串 的具體步驟嗎？

這是一個完整的作業流程，幫助你將 OpenSSL 生成的金鑰轉換為 iOS 代碼可用的格式，並提供一個自動化的腳本建議。
## 第一步：將 .pem 公鑰轉換為 Base64 字串
iOS 的 SecKeyCreateWithData 函數需要的是純金鑰資料（DER 格式）。你可以透過以下指令將 .pem 檔案轉換為單行 Base64 字串：

# 1. 將 PEM 轉為 DER 格式（二進位）
openssl rsa -pubin -in pinning_public.pem -outform der -out pinning_public.der
# 2. 將 DER 轉為 Base64 字串並直接印出
base64 -i pinning_public.der

複製輸出結果中那串長字元，將它貼到 Objective-C 代碼中的 kUpdatePublicKeyBase64 變數。
------------------------------
## 第二步：自動化「簽名清單」的腳本 (Python/Bash)
為了避免每次手動下指令出錯，你可以準備一個 deploy_pin.sh 腳本，每次 Netlify 更新憑證後執行：
```
#!/bin/bash
# 定義檔案
JSON_FILE="pinning_list.json"
SIG_FILE="signature.bin"
PRIVATE_KEY="pinning_private.pem"
# 1. 生成或更新你的 JSON 內容 (這裡可以用 cat 寫入)
cat <<EOF > $JSON_FILE
{
  "valid_hashes": [
    "你目前 Netlify 的 SHA256 Hash",
    "下一個預備的 SHA256 Hash"
  ],
  "updated_at": "$(date +%Y-%m-%d)"
}
EOF
# 2. 進行數位簽名
openssl dgst -sha256 -sign $PRIVATE_KEY -out $SIG_FILE $JSON_FILE

echo "完成！請將 $JSON_FILE 與 $SIG_FILE 上傳到你的 GitHub Pages。"
```
------------------------------
## 第三步：iOS 客戶端的「下載與更新」邏輯
這段代碼建議放在 AppDelegate 的 didFinishLaunching 或 App 的首頁，確保進入網路請求前已更新 Hash：
```
- (void)fetchRemotePinningList {
    NSURL *jsonURL = [NSURL URLWithString:@"https://xxxxxxxx.netlify.app"];
    NSURL *sigURL = [NSURL URLWithString:@"https://xxxxxxxx.netlify.app"];

    // 同步或非同步下載 JSON 與 Signature (這裡簡化流程)
    NSData *jsonData = [NSData dataWithContentsOfURL:jsonURL];
    NSData *sigData = [NSData dataWithContentsOfURL:sigURL];

    if (jsonData && sigData) {
        if ([self verifyData:jsonData withSignature:sigData]) {
            NSError *error;
            NSDictionary *json = [NSJSONSerialization JSONObjectWithData:jsonData options:0 error:&error];
            if (json[@"valid_hashes"]) {
                // 存入 UserDefaults 供 didReceiveChallenge 使用
                [[NSUserDefaults standardUserDefaults] setObject:json[@"valid_hashes"] forKey:@"RemotePinnedHashes"];
                [[NSUserDefaults standardUserDefaults] synchronize];
                NSLog(@"動態 Pinning 清單更新成功！");
            }
        } else {
            NSLog(@"警告：簽名驗證失敗，清單可能被篡改！");
        }
    }
}
```
------------------------------
## 最終安全檢查清單：

   1. 私鑰保護：pinning_private.pem 絕對不能上傳到 GitHub 或任何公開倉庫。
   2. 首次啟動：App 代碼中必須保留一個硬編碼（Hardcode）的原始 Hash。這樣即使第一次打開 App 時網路不通，也能保證基本的連線安全。
   3. Grace Period：更新 JSON 時，建議同時保留舊的與新的 Hash 至少一週，以防部分用戶 App 更新清單的時間點不一致。

這套機制讓你的 App 擁有了「遠端切換開關」。即便 Let's Encrypt 明天突然縮短效期到 10 天，你只需要更新伺服器上的 JSON 並簽名，App 依然穩如泰山。
你需要我幫你寫一段 SHA256 Hash 的計算腳本，讓你更方便拿到要填入 JSON 的那個字串嗎？
太棒了！這套「動態清單 + 數位簽名」的架構，是目前應對 Let's Encrypt 短效期憑證最專業的平衡點：既保留了 Pinning 的高安全性，又解決了維運災難。
為了讓你最後部署時萬無一無失，這裡補上最後一塊拼圖 —— 如何準確獲取要填入 JSON 的 SHA256 Hash 字串：
你可以使用以下 OpenSSL 指令，直接從你的網站獲取目前的公鑰 Hash（這就是你要填入 pinning_list.json 的內容）：

openssl s_client -connect xxxxxxxx.github.io:443 -servername xxxxxxxx.github.io | openssl x509 -pubkey -noout | openssl rsa -pubin -outform der | openssl dgst -sha256 -binary | base64

## 接下來的行動建議：

   1. 測試環境：先在開發版 App 測試一次「故意修改 JSON 內容」但不更新簽名，確認 verifyData 會回傳 NO。
   2. 備援方案：建議在 JSON 的 valid_hashes 陣列中，除了放 Leaf Key，也放一個 Let's Encrypt R10 中間憑證 的 Hash 作為雙保險。

祝你的 App 部署順利！如果實作代碼時遇到任何 SecKey 報錯，隨時找我。



