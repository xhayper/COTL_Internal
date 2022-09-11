# Save File

Cult of the Lamb use `JSON` structure to save and load the save data, it's encrypted using the `AES CBC` cryptographic algorithm.

The system can load both encrypted and unencrypted file,
if the file starts with "E" then the game will treat the file as encrypted file.

Backup will be created everytime the file is saved, it have a limit for 10 backup per file.

## Save File Structure

### Encrypted

| Byte       | 1                            | 2-17           | 18-33                 | 33+                     |
|------------|------------------------------|----------------|-----------------------|-------------------------|
| Field Name | Encrypted File Indicator     | Key            | IV                    | Data                    |
| Details    | Will always be character "E" | Encryption key | Initialization vector | The encrypted save data |

### UnEncrypted

| Byte       | 0+        |
|------------|-----------|
| Field Name | Data      |
| Details    | Save data |

## Example code

```csharp
using System.Security.Cryptography;
using Newtonsoft.Json;

namespace COTL_Internal;

public class SaveFile
{
    public static void Write<T>(string filePath, T data)
    {
        JsonSerializer jsonSerializer = null;
        StreamWriter streamWriter = null;
        CryptoStream cryptoStream = null;
        FileStream fileStream = null;
        Aes AES = null;

        byte[] Key = new byte[16];
        byte[] IV = new byte[16];

        try
        {
            using (var RNG = RandomNumberGenerator.Create())
            {
                RNG.GetBytes(Key);
                RNG.Dispose();
            }

            using (fileStream = new FileStream(filePath, FileMode.Create, FileAccess.Write))
            {
                using (AES = Aes.Create())
                {
                    IV = AES.IV;

                    fileStream.WriteByte((byte)69);
                    fileStream.Write(Key);
                    fileStream.Write(IV);

                    using (cryptoStream = new CryptoStream(fileStream, AES.CreateEncryptor(Key, IV), CryptoStreamMode.Write))
                    {
                        using (streamWriter = new StreamWriter(cryptoStream))
                        {
                            jsonSerializer = new JsonSerializer();
                            jsonSerializer.Serialize(streamWriter, data);
                        }
                    }
                }
            }
        }
        finally
        {
            AES?.Dispose();
            fileStream?.Dispose();
            cryptoStream?.Dispose();
            streamWriter?.Dispose();
        }
    }

    public static T Read<T>(string pathToFile)
    {
        string json = "";

        CryptoStream cryptoStream = null;
        StreamReader streamReader = null;
        FileStream fileStream = null;
        Aes AES = null;

        try
        {
            using (fileStream = new FileStream(pathToFile, FileMode.Open, FileAccess.Read))
            {
                byte headerByte = (byte)fileStream.ReadByte();

                if (headerByte == 69)
                {
                    byte[] Key = new byte[16];
                    byte[] IV = new byte[16];

                    fileStream.Read(Key, 0, 16);
                    fileStream.Read(IV, 0, 16);

                    using (AES = Aes.Create())
                    {
                        AES.Key = Key;
                        AES.IV = IV;

                        using (cryptoStream = new CryptoStream(fileStream, AES.CreateDecryptor(), CryptoStreamMode.Read))
                        {
                            using (streamReader = new StreamReader(cryptoStream))
                            {
                                json = streamReader.ReadToEnd();
                            }
                        }
                    }
                }
                else
                {
                    json = File.ReadAllText(pathToFile);
                }
            }

            return JsonConvert.DeserializeObject<T>(json);
        }
        finally
        {
            streamReader?.Dispose();
            cryptoStream?.Dispose();
            fileStream?.Dispose();
            AES?.Dispose();
        }
    }
}
```
