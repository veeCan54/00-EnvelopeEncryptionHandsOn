## Learn Envelope Encryption in 30 minutes - Hands On

Symmetric encryption is fast and with this type of encryption we are able to encrypt potentially large amounts of data. Envelope encryption is the practice of encrypting plain text data with a data key, and then encrypting the data key under another key. If these double encryptions sound a bit confusing to you, please read on. After you understand the concepts while doing this hands on, it should hopefully be clearer. If you are planning to create and use a KMS key on your own to use in your application, this guide should help. 
In AWS, when we select for example SSE - KMS, AWS does a lot of magic for us under the hood. 

> **Note:** This exercise will be done using the aws cli. We can check the status of keys using the AWS Admin console. I am making an assumption that you are using an IAM user whose credentials have been configured by running `aws configure`. I am also assuming that this IAM user has access to perform KMS operations e.g encrypt and decrypt. For the purpose of this demo, it is better to grant the IAM user all kms access. In other words, `kms.*`. Please ensure this before you start. For the demo, I am using a profile `iamadmin-general` and region `us-east-1` which is sent as an attribute in aws cli commands. If you have a default profile and have configured a default region you don’t have to send these two attributes in your aws cli commands. The commands used in the Hands On have been checked in [here](https://github.com/veeCan54/EnvelopeEncryptionHandsOn/blob/main/HandsOnInstructions.txt). Make sure values are substituted appropriately.

> **Cost:** Heads up that this exercise will create a new KMS key in AWS which has a small cost involved. From current official AWS documentation - Each AWS KMS key that you create in AWS KMS costs $1/month (prorated hourly). Assuming this exercise can be completed relatively quickly it should give you an idea of the approximate cost. Please make sure to schedule key deletion after the Hands On. This step is included at the very end of the exercise. 

## Envelope encryption involves 3 keys -  CMK or KEK, DEK and Encrypted DEK. 

CMK  - Customer Managed Key or KEK (Key Encryption Key). 
This is an AES 256 symmetric key that is created inside KMS and never leaves KMS, ever. 
CMK can only encrypt files up to 4 KB. When we create a KMS key via the admin console or via the cli, we are creating a CMK. 

Usually the actual data we need to encrypt is much bigger than 4 KB. If CMK can only encrypt files up to 4 KB in size, what is its purpose? 
CMK (KEK) is used to create what is called DEK - Data Encryption key. DEK is what encrypts and decrypts our data. The best practice is to have a distinct DEK for every piece of data for example objects in S3 buckets.
The diagram should illustrate this better.  
![Alt text](https://github.com/veeCan54/00-EnvelopeEncryptionHandsOn/blob/main/images/envEncryptionGraphic.png)
The bird graphic is courtesy of freepik.

**1. Let us create a CMK manually via aws cli :**

```sh 
aws kms create-key --description "CMK for Envelope Encryption Demo" --profile iamadmin-general --region "us-east-1"
```

In the response, The `KeyId` field uniquely identifies the key.

![Alt text](https://github.com/veeCan54/EnvelopeEncryptionHandsOn/blob/main/images/ceateKey.png)

**2. List the keys in your account with the following command and you should see the new KeyId.**

```sh 
aws kms list-keys --profile iamadmin-general --region us-east-1
```
![Alt text](https://github.com/veeCan54/EnvelopeEncryptionHandsOn/blob/main/images/listKeys.png)

If you look via the Admin Console you should see this key.

![Alt text](https://github.com/veeCan54/EnvelopeEncryptionHandsOn/blob/main/images/adminConsole1.png)

**3. Now to easily identify the key you can create an alias.**

```sh
aws kms create-alias --target-key-id "KeyID-Of-new-key" --alias-name alias/envEncryptionDemoCMK --profile iamadmin-general --region us-east-1
```

**4. After this you can list the alias you created and see that the TargetKeyId matches the Id of the CMK.**

```sh
aws kms list-aliases --query 'Aliases[?AliasName==`alias/envEncryptionDemoCMK`]' --profile iamadmin-general --region us-east-1
```
![Alt text](https://github.com/veeCan54/EnvelopeEncryptionHandsOn/blob/main/images/aliasAdded.png)

Now the Admin Console shows the alias as well. 

![Alt text](https://github.com/veeCan54/EnvelopeEncryptionHandsOn/blob/main/images/adminConsole2.png)

Now let’s see how to encrypt & decrypt with this CMK. 

**5. Identify the data to encrypt.**
Here I would like to encrypt the text file secret.txt. This file has the content “My super secret text.” Create the file with text in it.

![Alt text](https://github.com/veeCan54/EnvelopeEncryptionHandsOn/blob/main/images/secret.png)

**6. In order to encrypt and decrypt this file, we need the DEK. We do this using the generate-data-key api call. AES 256 is the algorithm we would like to use here.** 
> Note: If we use the `generate-data-key-without-plaintext` api call, we get only the CiphertextBlob back which can be used to derive the PlaintextKey later. Since this exercise is to demonstrate encryption, decryption and best practise, we're using the `generate-data-key` api call.

```sh
aws kms generate-data-key --key-id alias/envEncryptionDemoCMK --key-spec AES_256 --encryption-context project=envencr-demo --region us-east-1 --profile iamadmin-general
```
![Alt text](https://github.com/veeCan54/EnvelopeEncryptionHandsOn/blob/main/images/generateDataKey.png)

This returns two keys - 1. DEK and 2. Encrypted DEK. 

`Plaintext` field has the DEK value.

`CiphertextBlob` has the Encrypted DEK value. 

We need to understand here that CMK created earlier is used to encrypt the plaintext DEK to produce the Encrypted DEK.

>A few important points about PlainText DEK. 
DEK is used to encrypt and decrypt data. This data could be much larger than 4 KB. 
DEK should be discarded as soon as it is used. 
DEK should not be saved in a file and or committed to a repository. 
DEK should not be logged.
When using DEK in a program, it should only be in the stack, never in the heap. Heap dumps could potentially expose DEK.

Another attribute to note is the `--encryption-context` attribute in the generate-data-key api call. 
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

![Alt text](https://github.com/veeCan54/EnvelopeEncryptionHandsOn/blob/main/images/opensslEnc.png)

**10. As soon as this is done, discard the DEK.**

```sh
rm datakeyPlainText.txt
```

**11. Now that our secret is encrypted, delete unencrypted data.**

```sh
rm secret.txt 
```

![Alt text](https://github.com/veeCan54/EnvelopeEncryptionHandsOn/blob/main/images/rmSecret.png)

At this point, we have our encrypted data (secret-Encrypted.txt) along with the Encrypted DEK (encryptedDataKey.txt) 

Now our data is encrypted and safe. Say that later at some point, we want to decrypt this data. _(Since this is a demo, that later would have to be now)_. 

In order to decrypt, we need the DEK which we have discarded after encryption earlier.

We derive the plaintext data key using the encrypted data key. Remember that in order to do this, the IAM role should have the kms:decrypt privilege. `This is why it is important to control who has KMS encrypt and decrypt privileges.` As long as only select folks have this access, even if anyone stole our encrypted data key and the encrypted data, without being able to decrypt it, it will practically be useless for them. 

**12. To derive the plain text key, call the kms api like below:**

```sh
aws kms decrypt --encryption-context project=envencr-demo --ciphertext-blob fileb:////yourfilepath/encryptedDataKey.txt --profile iamadmin-general --region us-east-1
```

![Alt text](https://github.com/veeCan54/EnvelopeEncryptionHandsOn/blob/main/images/kmsDecrypt.png)

You will notice that this is the same PlainText DEK value. This is one of the 2 keys we got after Step 7 above. We discarded it after we encrypted the file secrets.txt earlier. 
We are going to save this file, use it for decryption and discard it after use.

**13. Save this file as datakeyPlainText.txt.**

```sh
echo "PlainTextKeyFromDecryptedKey" | base64 --decode > datakeyPlainText.txt
```

![Alt text](https://github.com/veeCan54/EnvelopeEncryptionHandsOn/blob/main/images/regenDataKeyPlainText.png)


**14. Now decrypt this file**  using `openssl enc -d`, `-in secret-Encrypted.txt` (input file) and `./secret-Decrypted.txt` (output file) 

```sh
openssl enc -d -aes256 -in secret-Encrypted.txt -k fileb:////yourfilepath/datakeyPlainText.txt > ./secret-Decrypted.txt
```

![Alt text](https://github.com/veeCan54/EnvelopeEncryptionHandsOn/blob/main/images/openSslDecrypt.png)

When you `cat` the decrypted file, your encrypted secret “My super secret text.” will be retreived as in the screen shot above. 

**15. Schedule Key deletion.**
It is possible to disable a key and enable it again however since we won't need this key anymore we can schedule a key deletion. 
The minimum waiting period for this is 7 days so that's what we will use. 
The ```schedule-key-deletion``` api call needs the ARN of the key, which we already have from our ```list-keys``` api call.

![Alt text](https://github.com/veeCan54/EnvelopeEncryptionHandsOn/blob/main/images/keyArn.png)

```sh
aws kms schedule-key-deletion \
    --key-id arn:aws:kms:us-east-1:123456789:key/yourKeyID \
    --pending-window-in-days 7 \
    --profile iamadmin-general \
    --region us-east-1

```
The response has KeyState, DeletionDate and PendingWindowInDays.

![Alt text](https://github.com/veeCan54/EnvelopeEncryptionHandsOn/blob/main/images/pendingDeletionCli.png)

The updated status shows on AWS admin console as well. 

![Alt text](https://github.com/veeCan54/EnvelopeEncryptionHandsOn/blob/main/images/pendingDeletion.png)

This key is scheduled to be deleted on DeletionDate. 
Now we have cleaned up after the Hands On.

## Summary:
AWS KMS generates, encrypts, and decrypts data keys. However, AWS KMS does not store, manage, or track your data keys, or perform cryptographic operations with data keys. You must use and manage data keys outside of AWS KMS. Plain text DEK should never be persisted. It should be recreated for every use and discarded immediately after use.  
It is important to ensure that only authorized parties have `kms:encrypt` and `kms:decrypt` privileges.  
Bird graphic courtesy of freepik <img src="https://github.com/veeCan54/00-EnvelopeEncryptionHandsOn/blob/main/images/freepic.png" width="70" height="10" />
 
