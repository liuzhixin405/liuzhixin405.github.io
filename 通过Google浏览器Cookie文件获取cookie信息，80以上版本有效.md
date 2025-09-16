# 通过Google浏览器Cookie文件获取cookie信息，80以上版本有效

**发布日期:** 2020-07-23  
**原文链接:** https://www.cnblogs.com/morec/p/13366946.html  
**标签:** 无

---

```

```
 public class ReadCookie
 {
 /// <summary>
 /// </summary>
 /// <param name="hostName"></param>
 /// <returns></returns>
 public static SeleCookie ReadCookies(string hostName,string cookiePath,string domain)
 {
 //测试用
 cookiePath = @"C: \Users\xxx\AppData\Local\Google\Chrome\User Data\Local State";
 domain = "%xxx.com";
 SeleCookie cc = new SeleCookie();
 if (hostName == null) throw new System.ArgumentNullException("hostName");

 var dbPath = System.Environment.GetFolderPath(System.Environment.SpecialFolder.LocalApplicationData) + @"\Google\Chrome\User Data\Default\Cookies";
 if (!System.IO.File.Exists(dbPath)) throw new System.IO.FileNotFoundException("Cant find cookie store", dbPath); // race condition, but i'll risk it

 var connectionString = "Data Source=" + dbPath + ";pooling=false";

 using (var conn = new System.Data.SQLite.SQLiteConnection(connectionString))
 using (var cmd = conn.CreateCommand())
 {
 conn.Close();
 cmd.CommandText = $"select * from cookies where host_key like '{domain}'";
 #region 查看cookie表
 SQLiteDataAdapter da = new SQLiteDataAdapter(cmd.CommandText, conn);
 DataSet ds = new DataSet();
 da.Fill(ds, "cookiestable");
 #endregion

 var prm = cmd.CreateParameter();
 prm.ParameterName = "hostName";
 prm.Value = hostName;
 cmd.Parameters.Add(prm);

 conn.Open();
 using (var reader = cmd.ExecuteReader())
 {
 string encKey = File.ReadAllText(cookiePath);
 encKey = JObject.Parse(encKey)["os_crypt"]["encrypted_key"].ToString();
 var decodedKey = System.Security.Cryptography.ProtectedData.Unprotect(Convert.FromBase64String(encKey).Skip(5).ToArray(), null, System.Security.Cryptography.DataProtectionScope.LocalMachine);
 try
 {
 var propInfo = typeof(SeleCookie).GetProperties().Where(x => x.IsDefined(typeof(CustomAttribute), false) && x.Name == reader[2].ToString() && !x.Name.Equals("Item"));
 while (reader.Read())
 {
 byte[] encryptedData = (byte[])reader[12];
 var _cookie = _decryptWithKey(encryptedData, decodedKey, 3);
 foreach (var prop in propInfo)
 {
 prop.SetValue(cc, _cookie ?? string.Empty);
 }
 }

 }
 finally
 {
 conn.Close();
 }

 }
 if (conn.State != ConnectionState.Closed)
 {
 conn.Close();
 }
 return cc;
 }

 }
 private static string _decryptWithKey(byte[] message, byte[] key, int nonSecretPayloadLength)
 {
 const int KEY_BIT_SIZE = 256;
 const int MAC_BIT_SIZE = 128;
 const int NONCE_BIT_SIZE = 96;

 if (key == null || key.Length != KEY_BIT_SIZE / 8)
 throw new ArgumentException(String.Format("Key needs to be {0} bit!", KEY_BIT_SIZE), "key");
 if (message == null || message.Length == 0)
 throw new ArgumentException("Message required!", "message");

 using (var cipherStream = new MemoryStream(message))
 using (var cipherReader = new BinaryReader(cipherStream))
 {
 var nonSecretPayload = cipherReader.ReadBytes(nonSecretPayloadLength);
 var nonce = cipherReader.ReadBytes(NONCE_BIT_SIZE / 8);
 var cipher = new GcmBlockCipher(new AesEngine());
 var parameters = new AeadParameters(new KeyParameter(key), MAC_BIT_SIZE, nonce);
 cipher.Init(false, parameters);
 var cipherText = cipherReader.ReadBytes(message.Length);
 var plainText = new byte[cipher.GetOutputSize(cipherText.Length)];
 try
 {
 var len = cipher.ProcessBytes(cipherText, 0, cipherText.Length, plainText, 0);
 cipher.DoFinal(plainText, len);
 }
 catch (InvalidCipherTextException)
 {
 return null;
 }
 return Encoding.Default.GetString(plainText);
 }
 }

 }
```

```

需要引用类库：

System.Security.Cryptography.ProtectedData

BouncyCastle

System.Data.SQLite.Core

Newtonsoft.Json

---

*本文档由博客园下载器自动生成*
