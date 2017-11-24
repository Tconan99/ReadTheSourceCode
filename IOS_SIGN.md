```
import SwiftyRSA

if (needSigned) {
    
    parametersConvertAll["client_app_id"] = headersConvert["client_app_id"]
    parametersConvertAll["client_device_id"] = headersConvert["client_device_id"]
    parametersConvertAll["access_token"] = headersConvert["access_token"]
    parametersConvertAll["timestamp"] = headersConvert["timestamp"]
    
    let paramArray = parametersConvertAll.sorted(by: {(dicLeft, dicRight) -> Bool in
        return String(describing: dicLeft.value).compare(String(describing: dicRight.value)).rawValue < 0
    })
    
    var paramValueString = ""
    for param in paramArray {
        paramValueString = paramValueString + String(describing: param.value);
    }
    
    let sha1 = Data(bytes: paramValueString.sha1Test())
    
    // ----------sign----------
    do {
        let publicKey = try PublicKey(pemNamed: "jcmp_m_public_key")
        
        let clear = ClearMessage(data: sha1)
        let sign = try clear.encrypted(with: publicKey, padding: [])
        
        // 转为16进制字符串
        let result = UnsafeMutablePointer<UInt8>.allocate(capacity: sign.data.count)
        sign.data.copyBytes(to: result, count: sign.data.count)
        
        
        let hash = NSMutableString()
        for i in 0..<sign.data.count {
            hash.appendFormat("%02x", Int(result[i]))
        }
        result.deallocate(capacity: sign.data.count)
        
        parametersConvert["sign"] = hash
        
    } catch {
        print(error.localizedDescription)
    }
    
}

```