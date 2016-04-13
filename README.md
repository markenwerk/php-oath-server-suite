# Oath Server Suite

[![Build Status](https://travis-ci.org/markenwerk/php-oath-server-suite.svg?branch=master)](https://travis-ci.org/markenwerk/php-oath-server-suite)
[![Code Climate](https://codeclimate.com/github/markenwerk/php-oath-server-suite/badges/gpa.svg)](https://codeclimate.com/github/markenwerk/php-oath-server-suite)
[![Latest Stable Version](https://poser.pugx.org/markenwerk/oath-server-suite/v/stable)](https://packagist.org/packages/markenwerk/oath-server-suite)
[![Total Downloads](https://poser.pugx.org/markenwerk/oath-server-suite/downloads)](https://packagist.org/packages/markenwerk/oath-server-suite)
[![License](https://poser.pugx.org/markenwerk/oath-server-suite/license)](https://packagist.org/packages/markenwerk/oath-server-suite)

A collection of classes to provide second factor authentication (Yubico OTP, TOTP, HOTP, GoogleAuthenticator) server-side.

For more information about Oath check out [https://openauthentication.org/](https://openauthentication.org/).

More information about TOTP (Time-based One-time Password Algorithm) can be found at [https://en.wikipedia.org/wiki/Time-based_One-time_Password_Algorithm](https://en.wikipedia.org/wiki/Time-based_One-time_Password_Algorithm).

More information about HOTP (HMAC-based One-time Password Algorithm) can be found at [https://en.wikipedia.org/wiki/HMAC-based_One-time_Password_Algorithm](https://en.wikipedia.org/wiki/HMAC-based_One-time_Password_Algorithm).

For more information about the Yubico OTP authentication mechanism read the „What is YubiKey OTP?“ article at [https://developers.yubico.com/OTP/](https://developers.yubico.com/OTP/).

## Installation

```{json}
{
   	"require": {
        "markenwerk/oath-server-suite": "~1.0"
    }
}
```

## Usage

### Autoloading and namesapce

```{php}  
require_once('path/to/vendor/autoload.php');
```

---

### Yubico OTP (YubiCloud)

To use Yubico OTP you need YubiCloud access. You can get free API credentials from [https://upgrade.yubico.com/getapikey/](https://upgrade.yubico.com/getapikey/).

#### Validating an Yubico OTP

```{php}
$otp = $_POST['otp'];
$userPublicId = $fetchedFromDatabaseOrSimilar;

$validator = new OathServerSuite\Validation\YubicoOtp\Validator('yubiCloudClientId', 'yubiCloudSecretKey');
try {
	$validator->validate($otp, $userPublicId);
	// Validation was success e.g. no exception was thrown.
} catch (OathServerSuite\Exception\NetworkException $exception) {
	// Accessing the YubiCloud webservice failed.
} catch (OathServerSuite\Exception\ValidationFailedException $exception) {
	if ($exception->getCode() == 1) {
		// The given OTP is not wellformed.
	} else if ($exception->getCode() == 2) {
		// The given OTP does not match the users public ID.
	} else if ($exception->getCode() == 3) {
		// The given OTP is not valid.
	}
}
```

---

### TOTP / HOTP (Google Authenticator style)

#### Sharing the key name and secret

To allow authentication the client and server has to share a secret. Usually the server dices a secret and displays it alltogether with the key name and the authentication mechanism as a QR code.

[Google Authenticator](https://en.wikipedia.org/wiki/Google_Authenticator) and some other applications and hardware items – like the [Yubikey](https://www.yubico.com/products/yubikey-hardware/) – do not follow the standard by expecting the secrets not as hexadecimal but as [Base32](https://en.wikipedia.org/wiki/Base32) encoded data.

##### TOTP (Time-based One-time Password Algorithm)

```{php}
use OathServerSuite\SecretSharing\SharedSecretQrCodeProvider\SharedSecretQrCodeProvider;
use OathServerSuite\SecretSharing\SharedSecretUrlEncoder\TotpBase32SharedSecretUrlEncoder;
use QrCodeSuite\QrEncode\QrEncoder;

// Initialize Oath URL encoder for TOTP (Time-based One-time Password Algorithm)
$contentEncoder = new TotpBase32SharedSecretUrlEncoder();

// Setting the key name
$keyName = 'Awesome Application';

// Setting a secret
// Attention: This is just an example value
// Use a random value of a proper length stored with your user credentials
$sharedSecret = openssl_random_pseudo_bytes(30);

// Getting the shared secret URL for usage wihtout QR code provision
$sharedSecretUrl = $contentEncoder->encode($keyName, $sharedSecret);

// Start QR code provision
// Initialize the QR code provider with Oath URL encoder for TOTP
$sharedSecretQrProvider = new SharedSecretQrCodeProvider(new TotpBase32SharedSecretUrlEncoder(), $keyName, $sharedSecret);

// Configure the QR code renderer for your needs
$sharedSecretQrProvider->getQrEncoder()
	->setLevel(QrEncoder::QR_CODE_LEVEL_LOW)
	->setTempDir('/path/to/a/writable/temp-dir');

// Persist the QR code PNG to the filesystem
$sharedSecretQrProvider->provideQrCode('/path/to/the/qrcode.png');
```

##### HOTP (HMAC-based One-time Password Algorithm)

```{php}
use OathServerSuite\SecretSharing\SharedSecretQrCodeProvider\SharedSecretQrCodeProvider;
use OathServerSuite\SecretSharing\SharedSecretUrlEncoder\HotpBase32SharedSecretUrlEncoder;
use QrCodeSuite\QrEncode\QrEncoder;

// Initialize Oath URL encoder for HOTP (HMAC-based One-time Password Algorithm)
$contentEncoder = new HotpBase32SharedSecretUrlEncoder();

// Setting the key name
$keyName = 'Awesome Application';

// Setting a secret
// Attention: This is just an example value
// Use a random value of a proper length stored with your user credentials
$sharedSecret = openssl_random_pseudo_bytes(30);

// Getting the shared secret URL for usage wihtout QR code provision
$sharedSecretUrl = $contentEncoder->encode($keyName, $sharedSecret);

// Start QR code provision
// Initialize the QR code provider with Oath URL encoder for HOTP
$sharedSecretQrProvider = new SharedSecretQrCodeProvider(new HotpBase32SharedSecretUrlEncoder(), $keyName, $sharedSecret);

// Configure the QR code renderer for your needs
$sharedSecretQrProvider->getQrEncoder()
	->setLevel(QrEncoder::QR_CODE_LEVEL_LOW)
	->setTempDir('/path/to/a/writable/temp-dir');

// Persist the QR code PNG to the filesystem
$sharedSecretQrProvider->provideQrCode('/path/to/the/qrcode.png');
```

---

### TOTP / HOTP (following the standard)

#### Sharing the key name and secret

##### TOTP (Time-based One-time Password Algorithm)

```{php}
use OathServerSuite\SecretSharing\SharedSecretQrCodeProvider\SharedSecretQrCodeProvider;
use OathServerSuite\SecretSharing\SharedSecretUrlEncoder\TotpSharedSecretUrlEncoder;
use QrCodeSuite\QrEncode\QrEncoder;

// Initialize Oath URL encoder for TOTP (Time-based One-time Password Algorithm)
$contentEncoder = new TotpSharedSecretUrlEncoder();

// Setting the key name
$keyName = 'Awesome Application';

// Setting a secret
// Attention: This is just an example value
// Use a random value of a proper length stored with your user credentials
$sharedSecret = openssl_random_pseudo_bytes(30);

// Getting the shared secret URL for usage wihtout QR code provision
$sharedSecretUrl = $contentEncoder->encode($keyName, $sharedSecret);

// Start QR code provision
// Initialize the QR code provider with Oath URL encoder for TOTP
$sharedSecretQrProvider = new SharedSecretQrCodeProvider(new TotpSharedSecretUrlEncoder(), $keyName, $sharedSecret);

// Configure the QR code renderer for your needs
$sharedSecretQrProvider->getQrEncoder()
	->setLevel(QrEncoder::QR_CODE_LEVEL_LOW)
	->setTempDir('/path/to/a/writable/temp-dir');

// Persist the QR code PNG to the filesystem
$sharedSecretQrProvider->provideQrCode('/path/to/the/qrcode.png');
```

##### HOTP (HMAC-based One-time Password Algorithm)

```{php}
use OathServerSuite\SecretSharing\SharedSecretQrCodeProvider\SharedSecretQrCodeProvider;
use OathServerSuite\SecretSharing\SharedSecretUrlEncoder\HotpSharedSecretUrlEncoder;
use QrCodeSuite\QrEncode\QrEncoder;

// Initialize Oath URL encoder for HOTP (HMAC-based One-time Password Algorithm)
$contentEncoder = new HotpSharedSecretUrlEncoder();

// Setting the key name
$keyName = 'Awesome Application';

// Setting a secret
// Attention: This is just an example value
// Use a random value of a proper length stored with your user credentials
$sharedSecret = openssl_random_pseudo_bytes(30);

// Getting the shared secret URL for usage wihtout QR code provision
$sharedSecretUrl = $contentEncoder->encode($keyName, $sharedSecret);

// Start QR code provision
// Initialize the QR code provider with Oath URL encoder for HOTP
$sharedSecretQrProvider = new SharedSecretQrCodeProvider(new HotpSharedSecretUrlEncoder(), $keyName, $sharedSecret);

// Configure the QR code renderer for your needs
$sharedSecretQrProvider->getQrEncoder()
	->setLevel(QrEncoder::QR_CODE_LEVEL_LOW)
	->setTempDir('/path/to/a/writable/temp-dir');

// Persist the QR code PNG to the filesystem
$sharedSecretQrProvider->provideQrCode('/path/to/the/qrcode.png');
```

## License

Oath Server Suite is under the MIT license.