### NTF身份头像的基本原理

![[Pasted image 20220812114423.png]]

NFT头像与常规头像有什么不同呢？

1.  𝐑𝐞𝐠𝐮𝐥𝐚𝐫 𝐩𝐫𝐨𝐟𝐢𝐥𝐞 pic
- Step 1: The user uploads a profile picture, and this request goes to the user service
- Step 2: The picture is stored in an object store, like Amazon S3. A URL is generated to visit the file
- Step 3: The picture’s metadata is stored in DB

2. NTF profile pic
- Step 1: To understand the process, we should know what smart contracts are. Smart contracts are programs deployed and stored on blockchains. They are self-executing when predetermined conditions are met. This is when an NFT is “minted.”

3. The inputs include the image file, name, and description. The mint function returns with the new token ID, metadata URI, and the NFT URI on IPFS (InterPlanetary File System). Let’s go through the outputs one by one.
4. Token ID is a unique ID for the NFT image. There is a dictionary in the smart contract that stores each token’s ID and its owner’s address. That’s why NFT is called “Non-Fungible Token.” Each image is assigned a unique ID.
5. Metadata and the profile pictures are stored on IPFS, a peer-to-peer network for storing and sharing data in a distributed filesystem. It’s an important extension for blockchains because it’s not possible to store all the data on blockchains.
6. IPFS leverages 𝐂𝐨𝐧𝐭𝐞𝐧𝐭 𝐀𝐝𝐝𝐫𝐞𝐬𝐬𝐢𝐧𝐠 to ensure the generated URI is linked to the file content. No one can replace or alter the file content without breaking the link. The generated metadata URIs are stored in another dictionary in the smart contract.
7. 
- Steps 2 and 3: Once the NFT is minted, we need to transfer it to the owner’s address. In blockchains, the address acts as a bank account number. We control the permissions via the private key stored in wallets like Metamask.
- Step 4: We can now authorize the Twitter page to have read-only access to the wallet. The service goes firstly to the smart contract and retrieves the metadata URI based on token ID. Then, it can load up the picture file from IPFS using the image URI in the metadata.

8. So, finally, we can see the cool avatar picture. You might notice that an NFT profile is more complex than a regular solution. But due to the use of smart contracts, blockchains, and IPFS content addressing, we can guarantee the integrity of the intellectual property.
9. Also, profile images become tradable assets or part of your personal identity.