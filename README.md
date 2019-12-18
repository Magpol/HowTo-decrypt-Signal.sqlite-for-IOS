By looking into the sourcecode of Signal on github i found the file "GRDBDatabaseStorageAdapter" to be of interest: 
https://github.com/signalapp/Signal-iOS/blob/e76583ac7d521e32f5de8ba4001f1847dea19c6e/SignalServiceKit/src/Storage/Database/GRDBDatabaseStorageAdapter.swift 

Signal is storing the DB-key in the keyvalue "GRDBDatabaseCipherKeySpec". The value is randomly generated when initialising the Database for the first time.
The Key and value are then stored in the keychain.

     private static let keyServiceName: String = "GRDBKeyChainService" 
     private static let keyName: String = "GRDBDatabaseCipherKeySpec" 

I also found out how Signal prepares and opens the DB:

     configuration.prepareDatabase = { (db: Database) in 
          let keyspec = try keyspec.fetchString() 
          try db.execute(sql: "PRAGMA key = \"\(keyspec)\"") 
          try db.execute(sql: "PRAGMA cipher_plaintext_header_size = 32") 

Using a decrypted keychain.plist i located the key "GRDBDatabaseCipherKeySpec" by searching for the keyname encoded in base64:

     user@computer:/Work/Signal$ echo -n GRDBDatabaseCipherKeySpec | base64 
     R1JEQkRhdGFiYXNlQ2lwaGVyS2V5U3BlYw== 

The value found under "v_data" is the value we are looking for:

     <key>v_Data</key> 
     <data> 
          VaVsbZDrjMKuVf3H+TUGCWcZMXn5zLGBxd9dnb/DYpAok+Q8Yw/zExRxMpGDfBee 
     </data> 

The value is the decoded using base64:

     user@computer:/Work/Signal$ echo -n VaVsbZDrjMKuVf3H+TUGCWcZMXn5zLGBxd9dnb/DYpAok+Q8Yw/zExRxMpGDfBee | base64 -d | xxd -p -c 48 
     55a56c6d90eb8cc2ae55fdc7f935060967193179f9ccb181c5df5d9dbfc362902893e43c630ff31314713291837c179e 

To open and decrypt the database i use sqlcipher. The version supplied via ubuntu failed in decrypting the db so i had to build sqlcipher from source.

      user@computer:/Work/Signal/sqlcipher/bld$ ./sqlcipher signal.sqlite 
      SQLCipher version 3.28.0 2019-04-16 19:49:53 
      Enter ".help" for usage hints. 
      sqlite> PRAGMA key="x'55a56c6d90eb8cc2ae55fdc7f935060967193179f9ccb181c5df5d9dbfc362902893e43c630ff31314713291837c179e'"; 
      sqlite> PRAGMA cipher_plaintext_header_size = 32; 
      sqlite> .tables 
      grdb_migrations                            
      keyvalue                                   
      media_gallery_items                      
      model_ExperienceUpgrade                    
      model_InstalledSticker                     
      model_KnownStickerPack                     
      model_OWSContactQuery                      
      model_OWSDevice            
      .... 
      sqlite> ATTACH DATABASE 'signal_decrypted.sqlite' AS signal_decrypted KEY ''; 
      sqlite> SELECT sqlcipher_export('signal_decrypted'); 
      sqlite> DETACH DATABASE signal_decrypted; 

The decrypted database is now found under signal_decrypted.sqlite.

