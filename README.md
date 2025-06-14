using System;
using System.Security.Cryptography;
using System.Text;

public static RSA LoadPrivateKeyFromPem(string pemKey)
{
    // Удаляем PEM-заголовки и декодируем Base64
    string base64 = pemKey
        .Replace("-----BEGIN PRIVATE KEY-----", "")
        .Replace("-----END PRIVATE KEY-----", "")
        .Replace("\n", "")
        .Trim();

    byte[] privateKeyDer = Convert.FromBase64String(base64);

    // Парсим DER-структуру PKCS#8 вручную (упрощенный вариант)
    RSAParameters rsaParams = ParsePkcs8PrivateKey(privateKeyDer);
    
    RSA rsa = RSA.Create();
    rsa.ImportParameters(rsaParams);
    return rsa;
}

private static RSAParameters ParsePkcs8PrivateKey(byte[] pkcs8Bytes)
{
    // Упрощенный парсинг PKCS#8 (для RSA ключей)
    // Структура PKCS#8: 
    //  0x30 (SEQUENCE) -> Version (INTEGER) + AlgorithmIdentifier (SEQUENCE) + PrivateKey (OCTET STRING)
    //  PrivateKey внутри — это PKCS#1 структура.

    int offset = 0;

    // Пропускаем внешний SEQUENCE (PKCS#8)
    if (pkcs8Bytes[offset++] != 0x30) 
        throw new ArgumentException("Invalid PKCS#8 format.");
    
    int outerSeqLength = ReadDerLength(pkcs8Bytes, ref offset);
    int endOffset = offset + outerSeqLength;

    // Пропускаем Version (обычно 0)
    if (pkcs8Bytes[offset++] != 0x02) 
        throw new ArgumentException("Invalid Version in PKCS#8.");
    
    int versionLength = ReadDerLength(pkcs8Bytes, ref offset);
    offset += versionLength;

    // Пропускаем AlgorithmIdentifier (OID для RSA)
    if (pkcs8Bytes[offset++] != 0x30) 
        throw new ArgumentException("Invalid AlgorithmIdentifier.");
    
    int algIdLength = ReadDerLength(pkcs8Bytes, ref offset);
    offset += algIdLength;

    // PrivateKey (OCTET STRING) -> внутри PKCS#1
    if (pkcs8Bytes[offset++] != 0x04) 
        throw new ArgumentException("Expected OCTET STRING for PrivateKey.");
    
    int privateKeyLength = ReadDerLength(pkcs8Bytes, ref offset);
    byte[] pkcs1Bytes = new byte[privateKeyLength];
    Array.Copy(pkcs8Bytes, offset, pkcs1Bytes, 0, privateKeyLength);

    // Теперь парсим PKCS#1 (RSA-specific)
    return ParsePkcs1PrivateKey(pkcs1Bytes);
}

private static RSAParameters ParsePkcs1PrivateKey(byte[] pkcs1Bytes)
{
    // Упрощенный парсинг PKCS#1 (RSA private key)
    int offset = 0;

    if (pkcs1Bytes[offset++] != 0x30) 
        throw new ArgumentException("Invalid PKCS#1 format.");
    
    int seqLength = ReadDerLength(pkcs1Bytes, ref offset);
    int endOffset = offset + seqLength;

    // Пропускаем Version (должен быть 0)
    if (pkcs1Bytes[offset++] != 0x02) 
        throw new ArgumentException("Invalid Version in PKCS#1.");
    
    int versionLength = ReadDerLength(pkcs1Bytes, ref offset);
    offset += versionLength;

    // Читаем параметры RSA (n, e, d, p, q, dp, dq, qi)
    RSAParameters rsaParams = new RSAParameters
    {
        Modulus = ReadDerInteger(pkcs1Bytes, ref offset),  // n
        Exponent = ReadDerInteger(pkcs1Bytes, ref offset), // e
        D = ReadDerInteger(pkcs1Bytes, ref offset),       // d
        P = ReadDerInteger(pkcs1Bytes, ref offset),       // p
        Q = ReadDerInteger(pkcs1Bytes, ref offset),       // q
        DP = ReadDerInteger(pkcs1Bytes, ref offset),      // dp
        DQ = ReadDerInteger(pkcs1Bytes, ref offset),      // dq
        InverseQ = ReadDerInteger(pkcs1Bytes, ref offset) // qi
    };

    return rsaParams;
}

// Вспомогательные методы для чтения DER-структуры
private static int ReadDerLength(byte[] data, ref int offset)
{
    int length = data[offset++];
    if ((length & 0x80) != 0)
    {
        int lengthBytes = length & 0x7F;
        length = 0;
        for (int i = 0; i < lengthBytes; i++)
        {
            length = (length << 8) | data[offset++];
        }
    }
    return length;
}

private static byte[] ReadDerInteger(byte[] data, ref int offset)
{
    if (data[offset++] != 0x02) 
        throw new ArgumentException("Expected INTEGER in DER.");
    
    int length = ReadDerLength(data, ref offset);
    byte[] value = new byte[length];
    Array.Copy(data, offset, value, 0, length);
    offset += length;

    // Удаляем ведущие нули (если есть)
    if (value.Length > 1 && value[0] == 0x00)
    {
        byte[] trimmed = new byte[value.Length - 1];
        Array.Copy(value, 1, trimmed, 0, trimmed.Length);
        return trimmed;
    }

    return value;
}
