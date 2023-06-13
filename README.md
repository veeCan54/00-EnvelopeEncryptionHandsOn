## Learn Envelope Encryption in 30 minutes - Hands On

Symmetric encryption is fast and with this type of encryption we are able to encrypt potentially large amounts of data. Envelope encryption is the practice of encrypting plain text data with a data key, and then encrypting the data key under another key. If these double encryptions sound a bit confusing to you, please read on. After you understand the concepts while doing this hands on, it should hopefully be clearer. If you are planning to create and use a KMS key on your own to use in your application, this guide should help. 
In AWS, when we select for example SSE - KMS, aws does a lot of magic for us under the hood. 

> **Note:** I try to use the aws cli as much as possible so that is what I am using here. 
This demo will be completely manual. I am making an assumption that you are using an IAM user whose credentials have been configured by running `aws configure`. I am also assuming that this user has access to perform KMS operations e.g encrypt and decrypt.  For the purpose of this demo, it is better to grant the IAM user all kms access. In other words, `kms.*`. Please ensure this before you start. For the demo, I am using a profile `iamadmin-general` and region `us-east-1` which is sent as an attribute in aws cli commands. If you have a default profile and have configured a default region you don’t have to send these two attributes in your aws cli commands. The commands used in the Hands-On have been checked in [here](https://github.com/veeCan54/EnvelopeEncryptionHandsOn/blob/main/HandsOnInstructions.txt). Make sure values are substituted appropriately.

## Envelope encryption involves 3 keys -  CMK or KEK, DEK and Encrypted DEK. 

CMK  - Customer Managed Key or KEK (Key Encryption Key). 
This is an AES 256 symmetric key that is created inside KMS and never leaves KMS, ever. 
CMK can only encrypt files up to 4 KB. When we create a KMS key via the admin console or via the cli, we are creating a CMK. 

Usually the actual data we need to encrypt is much bigger than 4 KB. If CMK can only encrypt files up to 4 KB in size, what is its purpose? 
CMK (KEK) is used to create what is called DEK - Data Encryption key. DEK is what encrypts and decrypts our data. The best practice is to have a distinct DEK for every piece of data for example objects in S3 buckets.

![Alt text](https://github.com/veeCan54/TestMBPro/blob/main/images/image10.png)

The diagram should illustrate this better. This image is courtesy of Adrian Cantrill who creates amazing learning content at https://learn.cantrill.io/

**1. Let us create a CMK manually via aws cli :**

```sh 
aws kms create-key --description "CMK for Envelope Encryption Demo" --profile iamadmin-general --region "us-east-1"
```

In the response, The `KeyId` field uniquely identifies the key.

![Alt text](https://github.com/veeCan54/TestMBPro/blob/main/images/ceateKey.png)

**2. List the keys in your account with the following command and you should see the new KeyId.**

```sh 
aws kms list-keys --profile iamadmin-general --region us-east-1
```
![Alt text](https://github.com/veeCan54/TestMBPro/blob/main/images/listKeys.png)

If you look via the Admin Console you should see this key.

![Alt text](https://github.com/veeCan54/TestMBPro/blob/main/images/adminConsole1.png)

**3. Now to easily identify the key you can create an alias.**

```sh
aws kms create-alias --target-key-id "KeyID-Of-new-key" --alias-name alias/envEncryptionDemoCMK --profile iamadmin-general --region us-east-1
```

**4. After this you can list the alias you created and see that the TargetKeyId matches the Id of the CMK.**

```sh
aws kms list-aliases --query 'Aliases[?AliasName==`alias/envEncryptionDemoCMK`]' --profile iamadmin-general --region us-east-1
```
![Alt text](https://github.com/veeCan54/TestMBPro/blob/main/images/aliasAdded.png)

Now the Admin Console shows the alias as well. 

![Alt text](https://github.com/veeCan54/TestMBPro/blob/main/images/adminConsole2.png)

Now let’s see how to encrypt & decrypt with this CMK. 

**5. Identify the data to encrypt.**
Here I would like to encrypt the text file secret.txt. This file has the content “My super secret text.” Create the file with text in it.

![Alt text](https://github.com/veeCan54/TestMBPro/blob/main/images/secret.png)

**6. In order to encrypt and decrypt this file, we need the DEK. We do this using the generate-data-key api call. AES 256 is the algorithm we would like to use here.** 

```sh
aws kms generate-data-key --key-id alias/envEncryptionDemoCMK --key-spec AES_256 --encryption-context project=envencr-demo --region us-east-1 --profile iamadmin-general
```
![Alt text](https://github.com/veeCan54/TestMBPro/blob/main/images/generateDataKey.png)


This returns two keys - 1. DEK and 2. Encrypted DEK. 

`PlainText` field has the DEK value.

`CiphertextBlob` has the Encrypted DEK value. 

We need to understand here that CMK created earlier is used to encrypt the plaintext DEK to produce the Encrypted DEK.

>A few important points about PlainText DEK. 
DEK is used to encrypt and decrypt data. This data could be much larger than 4 KB. 
DEK should be discarded as soon as it is used. 
DEK should not be saved in a file and or committed to a repository. 
DEK should not be logged.
When using DEK in a program, it should only be in the stack, never in the heap. Heap dumps could potentially expose DEK.

Another attribute to note is the `--encryption-context` attribute in the generate-data-api call. 
>Encryption context is a set of non-secret key value pairs that can contain additional contextual information about the data. It is optional. If provided, aws uses encryption context to provide authenticated encryption in order to ensure data integrity and confidentiality. If the encryption context is provided for encryption, then it should be provided for decryption as well. One could say that even if your encrypted keys are compromised, without the encryption context they cannot be used for decryption. The other real reason being kms:decrypt privilege in IAM. Without kms:decrypt privilege, the encrypted key cannot be used to derive the plain text key in order to decrypt the data. 

Now let’s proceed. Let’s save DEK and Encrypted DEK values in order to proceed with encryption and decryption. 

**7. Save DEK. Copy the value of Plaintext and save it as datakeyPlainText.txt. Copy it all the way from beginning to ending “”.**

```sh 
echo "PlainTextValueFromStep6" | base64 --decode > datakeyPlainText.txt
```

**8. Save Encrypted DEK. Copy the value of CipherTextBlob and save it as encryptedKeytxt. Copy it all the way from beginning to ending “”.**
```sh 

echo "CipherTextBlobValueFromStep6" | base64 --decode > encryptedDataKey.txt
```

**9. Now we can encrypt our secret file using the Plaintext data key, DEK.** 
We will be discarding DEK  after use. We use `openssl enc -e`, `-in secret.txt` (input file) and `-out secret-Encrypted.txt` (output file)

```sh 
openssl enc -e -aes256 -in secret.txt -out secret-Encrypted.txt -k fileb:////yourfilepath/datakeyPlainText.txt
```

This creates the secret-Encrypted.txt file. 

![Alt text](https://github.com/veeCan54/TestMBPro/blob/main/images/opensslEnc.png)

**10. As soon as this is done, discard the DEK.**

```sh
rm datakeyPlainText.txt
```

**11. Now that our secret is encrypted, delete unencrypted data.**

```sh
rm secret.txt 
```

![Alt text](https://github.com/veeCan54/TestMBPro/blob/main/images/rmSecret.png)

At this point, we have our encrypted data (secret-Encrypted.txt) along with the Encrypted DEK (encryptedDataKey.txt) 

Now your data is encrypted and safe. Say that later at some point, you want to decrypt this data. For demo purposes, that later would have to be now. 

In order to decrypt, we need the DEK which we have discarded after encryption earlier.

We derive the plaintext data key using the encrypted data key. Remember that in order to do this, the IAM role should have the kms:decrypt privilege. `This is why it is important to control who has KMS encrypt and decrypt privileges.` As long as only select folks have this access, even if anyone stole our encrypted data key and the encrypted data, without being able to decrypt it, it will practically be useless for them. 

**12. To derive the plain text key, call the kms api like below:**

```sh
aws kms decrypt --encryption-context project=envencr-demo --ciphertext-blob fileb:////yourfilepath/encryptedDataKey.txt --profile iamadmin-general --region us-east-1
```

![Alt text](https://github.com/veeCan54/TestMBPro/blob/main/images/kmsDecrypt.png)

You will notice that this is the same PlainText DEK value. This is one of the 2 keys we got after Step 7 above. We discarded it after we encrypted the file secrets.txt earlier. 
We are going to save this file, use it for decryption and discard it after use.

**13. Save this file as datakeyPlainText.txt.**

```sh
echo "PlainTextKeyFromDecryptedKey" | base64 --decode > datakeyPlainText.txt
```

![Alt text](https://github.com/veeCan54/TestMBPro/blob/main/images/regenDataKeyPlainText.png)


Now decrypt this file using `openssl enc -d`, `-in secret-Encrypted.txt` (input file) and `./secret-Decrypted.txt` (output file) 

```sh
openssl enc -d -aes256 -in secret-Encrypted.txt -k fileb:////yourfilepath/datakeyPlainText.txt > ./secret-Decrypted.txt
```

![Alt text](https://github.com/veeCan54/TestMBPro/blob/main/images/openSslDecrypt.png)

When you `cat` the decrypted file, your encrypted secret “My super secret text.” will be retreived as in the screen shot above. 

**Summary:**
KMS does the encryption and decryption of DEK, your application does the encryption and decryption of data using the DEK. 
Plain text DEK should never be persisted. It should be recreated for every use and discarded immediately after use. 
It is important to ensure that only authorized parties have `kms:encrypt` and `kms:decrypt` privileges. 
