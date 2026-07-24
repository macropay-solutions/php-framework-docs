# Encryption

- [Configuration](#configuration)
- [Basic Usage](#basic-usage)

<a name="configuration"></a>
## Configuration

Before using Framework's encrypter, you should set the `APP_KEY` option of your `.env` file to a 32 character, random string. If this value is not properly set, all values encrypted by Framework will be insecure.

<a name="gracefully-rotating-encryption-keys"></a>
## Gracefully Rotating Encryption Keys
If you change your application's encryption key, all authenticated user sessions will be logged out of your application. This is because every cookie, including session cookies, is encrypted by Framework. In addition, it will no longer be possible to decrypt any data that was encrypted with your previous encryption key.

To mitigate this issue, Framework allows you to configure your previous encryption keys and ciphers in your application's APP_PREVIOUS_KEYS_CIPHERS_MAP_JSON environment variable as json. This variable may contain a json map with previous key mapped to its cipher:

APP_KEY="base64:J63qRTDLub5NuZvP+kb8YIorGS6qFYHKVo6u7179stY="
APP_PREVIOUS_KEYS_CIPHERS_MAP_JSON={"base64:2nLsGFGzyoae2ax3EF2Lyq/hH6QghBGLIq5uL+Gp8/w=":"AES-256-CBC", ...}

When you set this environment variable, Framework will always use the "current" encryption key when encrypting values. However, when decrypting values, Framework will first try the current key, and if decryption fails using the current key, Framework will try all previous keys until one of the keys is able to decrypt the value.

This approach to graceful decryption allows users to keep using your application uninterrupted even if your encryption key is rotated.


<a name="basic-usage"></a>
## Basic Usage

#### Encrypting A Value

All encrypted values are encrypted using OpenSSL and the `AES-256-CBC` cipher. Furthermore, all encrypted values are signed with a message authentication code (MAC) to detect any modifications to the encrypted string.

For example, we may use the `encrypt` method to encrypt a secret and store it on an [Obvious model](/obvious):

	<?php

	namespace App\Http\Controllers;

    use App\Models\User;
    use MacropaySolutions\Kernel\Http\Request;
    use MacropaySolutions\Kernel\Http\Response;

	class UserController extends Controller
	{
		/**
		 * Store a secret message for the user.
		 */
		public function storeSecret(Request $request, int|string $id): Response
		{
			$user = User::query()->findOrFail($id);

			$user->fill([
				'secret' => \app('encrypter')->encrypt($request->getFiltered('secret'))
			])->save();

            return \response()->json($user->toArray());
		}
	}

#### Decrypting A Value

If the value can not be properly decrypted, such as when the MAC is invalid, an `MacropaySolutions\Kernel\Contracts\Encryption\DecryptException` will be thrown:

	use MacropaySolutions\Kernel\Contracts\Encryption\DecryptException;

	try {
		$decrypted = \app('encrypter')->decrypt($encryptedValue);
	} catch (DecryptException $e) {
		//
	}
