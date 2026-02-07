---
description: Security features for Seeed Studio XIAO ESP32C3
title: Security Features
keywords:
- xiao
- esp32c3
- security
- flash encryption
- secure boot
image: https://files.seeedstudio.com/wiki/wiki-platform/S-tempor.png
slug: /XIAO_ESP32C3_Security
last_update:
  date: 02/07/2026
  author: Claude
---

# XIAO ESP32C3 Security Features

The ESP32-C3 includes hardware-accelerated security features for protecting firmware, data, and communications. This guide covers flash encryption, secure boot, and cryptographic capabilities relevant to production IoT deployments.

## Security Hardware Overview

| Feature | Description |
|---|---|
| Flash Encryption | AES-128/256 transparent encryption of external flash |
| Secure Boot v2 | RSA-PSS signature verification of bootloader and app |
| AES Accelerator | Hardware AES-128/256 for data encryption |
| SHA Accelerator | Hardware SHA-1/224/256 |
| RSA Accelerator | Up to 3072-bit key operations |
| HMAC Module | Key-derived HMAC generation (keys stored in eFuse) |
| Digital Signature | Hardware private-key operations without exposing keys |
| TRNG | True Random Number Generator seeded by hardware noise |
| eFuse | 4096 bits of one-time programmable secure storage |

## Flash Encryption

Flash encryption ensures the contents of the external SPI flash are encrypted at rest. Even if an attacker physically reads the flash chip, the data is unintelligible.

### How It Works

- The ESP32-C3 generates or uses a pre-programmed AES key stored in eFuse
- All reads from flash are transparently decrypted by hardware
- All writes to flash are transparently encrypted
- The AES key in eFuse can be made read-protected so software cannot extract it

### Development Mode vs Release Mode

| Aspect | Development Mode | Release Mode |
|---|---|---|
| Re-flashing | Allowed (limited times) | Disabled after setup |
| UART download | Enabled | Disabled |
| Key access | Software-readable | Read-protected in eFuse |
| Use case | Testing encryption | Production deployment |

:::caution
**Release mode flash encryption is IRREVERSIBLE.** Once enabled, you cannot revert to plaintext flash. Always test in Development mode first.
:::

### Enabling Flash Encryption (ESP-IDF)

Flash encryption is configured through ESP-IDF's `menuconfig`. It is **not available** directly through Arduino IDE.

```bash
# In your ESP-IDF project
idf.py menuconfig
```

Navigate to: **Security features > Enable flash encryption on boot**

Select mode:
- **Development** for testing
- **Release** for production (irreversible)

```bash
# Build and flash with encryption enabled
idf.py build
idf.py flash
```

On first boot, the bootloader will:
1. Generate an AES-256 key (if not pre-loaded)
2. Burn the key into eFuse
3. Encrypt the flash contents in-place
4. Set eFuse flags to enable encryption

### Encrypted OTA Updates

When flash encryption is enabled, OTA updates still work -- the new firmware is encrypted as it is written to flash. No changes to your OTA code are needed; the hardware handles encryption transparently.

## Secure Boot v2

Secure Boot ensures only authenticated (digitally signed) firmware can run on the device. It prevents unauthorized firmware from being flashed.

### How It Works

1. You generate an RSA-3072 signing key pair
2. The public key hash is burned into eFuse
3. The bootloader verifies the app image signature on every boot
4. If verification fails, the chip refuses to boot

### Setting Up Secure Boot (ESP-IDF)

```bash
# Generate a signing key (keep this safe!)
espsecure.py generate_signing_key --version 2 secure_boot_signing_key.pem
```

In `menuconfig`:
- **Security features > Enable hardware Secure Boot in bootloader**
- **Secure Boot v2 > RSA-based Secure Boot**
- Set the signing key path

```bash
idf.py build
idf.py flash
```

:::caution
**Secure Boot is IRREVERSIBLE once burned.** If you lose your signing key, you can never update the device firmware again.
:::

### Signing Key Management

- **Never** commit signing keys to version control
- Store backup copies in a hardware security module (HSM) or secure vault
- Use a CI/CD pipeline to sign firmware during builds
- Consider using multiple signing keys for key rotation

## Using the Hardware Crypto Accelerators

### AES Encryption

Use the hardware AES accelerator for fast data encryption:

```cpp
#include <mbedtls/aes.h>

void encryptData() {
  mbedtls_aes_context aes;
  mbedtls_aes_init(&aes);

  // 128-bit key
  unsigned char key[16] = {
    0x00, 0x01, 0x02, 0x03, 0x04, 0x05, 0x06, 0x07,
    0x08, 0x09, 0x0A, 0x0B, 0x0C, 0x0D, 0x0E, 0x0F
  };

  unsigned char plaintext[16]  = "Hello, ESP32C3!";
  unsigned char ciphertext[16] = {0};
  unsigned char decrypted[16]  = {0};

  // Encrypt
  mbedtls_aes_setkey_enc(&aes, key, 128);
  mbedtls_aes_crypt_ecb(&aes, MBEDTLS_AES_ENCRYPT, plaintext, ciphertext);

  // Decrypt
  mbedtls_aes_setkey_dec(&aes, key, 128);
  mbedtls_aes_crypt_ecb(&aes, MBEDTLS_AES_DECRYPT, ciphertext, decrypted);

  Serial.printf("Decrypted: %s\n", decrypted);

  mbedtls_aes_free(&aes);
}
```

### SHA-256 Hashing

```cpp
#include <mbedtls/sha256.h>

void computeHash() {
  const char* message = "Hello, ESP32C3!";
  unsigned char hash[32];

  mbedtls_sha256_context sha256;
  mbedtls_sha256_init(&sha256);
  mbedtls_sha256_starts(&sha256, 0);  // 0 = SHA-256
  mbedtls_sha256_update(&sha256, (const unsigned char*)message, strlen(message));
  mbedtls_sha256_finish(&sha256, hash);
  mbedtls_sha256_free(&sha256);

  Serial.print("SHA-256: ");
  for (int i = 0; i < 32; i++) {
    Serial.printf("%02x", hash[i]);
  }
  Serial.println();
}
```

### True Random Number Generator (TRNG)

The ESP32-C3's hardware TRNG provides cryptographically secure random numbers:

```cpp
void setup() {
  Serial.begin(115200);

  // Generate cryptographically secure random numbers
  uint32_t random1 = esp_random();
  uint32_t random2 = esp_random();

  Serial.printf("Random 1: 0x%08X\n", random1);
  Serial.printf("Random 2: 0x%08X\n", random2);

  // Fill a buffer with random bytes
  uint8_t buffer[32];
  esp_fill_random(buffer, sizeof(buffer));

  Serial.print("Random bytes: ");
  for (int i = 0; i < 32; i++) {
    Serial.printf("%02X", buffer[i]);
  }
  Serial.println();
}

void loop() {}
```

## TLS/SSL for Secure Communication

### HTTPS Client

Use TLS to secure data transmission:

```cpp
#include <WiFiClientSecure.h>

WiFiClientSecure client;

void setup() {
  Serial.begin(115200);
  WiFi.begin("SSID", "password");
  while (WiFi.status() != WL_CONNECTED) delay(500);

  // Option 1: Use a root CA certificate (recommended)
  // client.setCACert(root_ca_cert);

  // Option 2: Skip certificate verification (NOT for production)
  client.setInsecure();

  if (client.connect("api.example.com", 443)) {
    client.println("GET /data HTTP/1.1");
    client.println("Host: api.example.com");
    client.println("Connection: close");
    client.println();

    while (client.connected()) {
      String line = client.readStringUntil('\n');
      Serial.println(line);
    }
  }

  client.stop();
}

void loop() {}
```

### Embedding Root CA Certificates

For production, always verify the server's certificate:

```cpp
const char* root_ca = R"EOF(
-----BEGIN CERTIFICATE-----
MIIDdzCCAl+gAwIBAgIEAgAAuTANBgkqhkiG9w0BAQUFADBaMQswCQYD
...your root CA certificate here...
-----END CERTIFICATE-----
)EOF";

void setup() {
  // ...
  client.setCACert(root_ca);
  // Now connections will verify the server's certificate chain
}
```

## Security Best Practices for IoT Deployment

1. **Enable flash encryption** in release mode for production devices
2. **Enable secure boot** to prevent unauthorized firmware
3. **Use TLS/HTTPS** for all network communication
4. **Store credentials** in NVS with encryption, not hardcoded in source
5. **Use the TRNG** for all cryptographic operations (never use `random()`)
6. **Disable UART download** mode in production (via eFuse)
7. **Keep firmware updated** with OTA -- see [OTA Updates](/XIAO_ESP32C3_OTA_Updates)
8. **Minimize attack surface** -- disable unused Wi-Fi/BLE features

## Further Reading

- [Architecture Overview](/XIAO_ESP32C3_Architecture) - Security hardware block details
- [OTA Firmware Updates](/XIAO_ESP32C3_OTA_Updates) - Secure firmware update process
- [WiFi Usage](/XIAO_ESP32C3_WiFi_Usage) - Wireless connectivity setup
