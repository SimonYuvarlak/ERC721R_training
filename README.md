# ERC721R_training
This is the explanation of the ERC721R developed by exo-digital-lab

# This is an example of how to implement ERC721R standart to your code
```solidity
// SPDX-License-Identifier: MIT

pragma solidity >=0.8.0;

import "./Ownable.sol";

contract Refundable is Ownable {
    uint256 public constant refundPeriod = 45 days; //Period to refund the token
    uint256 public refundEndTime; //Deadline of refunding
    address public refundAddress; //Address where will be the nft going to be refunded

    constructor() Refundable("ERC721RExample", "ERC721R") {
        refundAddress = msg.sender; //Refund address is initialized as the intantiator address
        toggleRefundCountdown();
    }

    //This function checks whether the refund deadline has been passed or not
    function isRefundGuaranteeActive() public view returns (bool) {
        return (block.timestamp <= refundEndTime); //block.timestamp returns the date of the creation and compares it with the refundtime
    }

    //Refund time end getter
    function getRefundGuaranteeEndTime() public view returns (uint256) {
        return refundEndTime;
    }

    //This function can only be called from outside of the contract
    function refund(uint256[] calldata tokenIds) external {
        require(isRefundGuaranteeActive(), "Refund expired"); //Check whether the user can still refund the nft

        //This loop checks all the ids minted on the contract
        //If there is a token of the user, who is the one who called function, the nft is returned to the refund address which is the initator of the contract
        for (uint256 i = 0; i < tokenIds.length; i++) {
            uint256 tokenId = tokenIds[i]; //Gets the token id from the array of nfts to compare with the function caller (msg.sender) given id number
            require(msg.sender == ownerOf(tokenId), "Not token owner"); //Checks if the msg.sender has the nft
            transferFrom(msg.sender, refundAddress, tokenId); //Refunds the nft to the contract owner
        }

        uint256 refundAmount = tokenIds.length * mintPrice; //Generates the price
        Address.sendValue(payable(msg.sender), refundAmount); //Returns the price of the nft (gas fee has lost)
    }

    //Starts the refund countdown by adding the days to the block.timestamp which is the -new Date()- in js
    //Only callable by the owner of the contract
    function toggleRefundCountdown() public onlyOwner {
        refundEndTime = block.timestamp + refundPeriod;
    }

    //Setter for the refund address and is only callable by the owner of the contract
    function setRefundAddress(address _refundAddress) external onlyOwner {
        refundAddress = _refundAddress;
    }

}
```

# This is an example ERC721R contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "erc721a/contracts/ERC721A.sol";
import "@openzeppelin/contracts/utils/cryptography/MerkleProof.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

//This contract is based on the ERC721 (non fungible token standart on Ethereum network) standart

contract ERC721RExample is ERC721A, Ownable {
    uint256 public constant maxMintSupply = 8000; //Maximum mintable nft count
    uint256 public constant mintPrice = 0.1 ether; //Cost of minting a nft
    uint256 public constant refundPeriod = 45 days; //Days that user can refund a nft (gas fee will be lost)

    // Sale Status
    bool public publicSaleActive; //Whether there will be public sale or not
    bool public presaleActive; //Whether there will be a presale or not
    uint256 public refundEndTime; //When will refund time will end

    address public refundAddress; //The address where the nfts will be refunded
    uint256 public constant maxUserMintAmount = 5; //A user can mint maximum of 5 nfts
    bytes32 public merkleRoot; //Merkle root

    mapping(uint256 => bool) public hasRefunded; // users can search if the NFT has been refunded
    mapping(uint256 => bool) public isOwnerMint; // if the NFT was freely minted by owner

    string private baseURI; //Base URI

    constructor() ERC721A("ERC721RExample", "ERC721R") {
        refundAddress = msg.sender; //Refund address is set to the initializor's address
        toggleRefundCountdown(); //Starting the refund time
    }

    //Function to mint nft before the original sale, based on the proof and whether the user on the list or not
    //Can only be called from outside of the contract
    function preSaleMint(uint256 quantity, bytes32[] calldata proof)
        external
        payable
    {
        require(presaleActive, "Presale is not active"); //Checks whether the presale is active or not
        require(msg.value == quantity * mintPrice, "Value"); //If the function is embedded with the enough amount for minting
        require(
            _isAllowlisted(msg.sender, proof, merkleRoot),
            "Not on allow list"
        ); //Checks whether the user is in the list, based on the function caller's address, its proof and the merkleroot
        require(
            _numberMinted(msg.sender) + quantity <= maxUserMintAmount,
            "Max amount"
        ); //User mint number limitation
        require(_totalMinted() + quantity <= maxMintSupply, "Max mint supply");

        _safeMint(msg.sender, quantity); //Minting function
    }

    //Function to mint a nft, during the normal sale process
    //Can only be called from outside of the function
    function publicSaleMint(uint256 quantity) external payable {
        require(publicSaleActive, "Public sale is not active"); //Checks whether the public sale has started
        require(msg.value >= quantity * mintPrice, "Not enough eth sent"); //Does the message has enough funds
        require(
            _numberMinted(msg.sender) + quantity <= maxUserMintAmount,
            "Over mint limit"
        ); //Checking if the user reached the limit
        require(
            _totalMinted() + quantity <= maxMintSupply,
            "Max mint supply reached"
        ); //Does the contract reach the max mint limit

        _safeMint(msg.sender, quantity); //Minting
    }

    //Function for owner to mint
    //Can only be called from outside of the contract
    //Only owner can call this funciton
    function ownerMint(uint256 quantity) external onlyOwner {
        require(
            _totalMinted() + quantity <= maxMintSupply,
            "Max mint supply reached"
        ); //Does the contract reach the maximum minting limit
        _safeMint(msg.sender, quantity); //Mint the token

        //This loops check changes the values to true for the owner minting mapping that is defines on top
        for (uint256 i = _currentIndex - quantity; i < _currentIndex; i++) {
            isOwnerMint[i] = true;
        }
    }

    //Function to refund tokens to user if needed
    //Can only be called from outside of the function
    function refund(uint256[] calldata tokenIds) external {
        require(isRefundGuaranteeActive(), "Refund expired"); //Is refund still active

        //This loops finds the token based on the id
        for (uint256 i = 0; i < tokenIds.length; i++) {
            uint256 tokenId = tokenIds[i]; //Assigning token to the list item
            require(msg.sender == ownerOf(tokenId), "Not token owner"); //If this is the token of the owner
            require(!hasRefunded[tokenId], "Already refunded"); //Is the token is already refunded
            require(
                !isOwnerMint[tokenId],
                "Freely minted NFTs cannot be refunded"
            ); //Owner cannot refund
            hasRefunded[tokenId] = true; //Mark token as refunded
            transferFrom(msg.sender, refundAddress, tokenId); //Refund token to the contract creator
        }

        uint256 refundAmount = tokenIds.length * mintPrice; //Get the amount to be refunded
        Address.sendValue(payable(msg.sender), refundAmount); //Refund the amount to the owner
    }

    //When the refund is ending
    function getRefundGuaranteeEndTime() public view returns (uint256) {
        return refundEndTime;
    }

    //Does refund time passed
    function isRefundGuaranteeActive() public view returns (bool) {
        return (block.timestamp <= refundEndTime);
    }

    //Function to withdraw money
    //Only owner can call this function
    //This function can only be called from outside of the contract
    function withdraw() external onlyOwner {
        require(block.timestamp > refundEndTime, "Refund period not over"); //Did the refund period end
        uint256 balance = address(this).balance; //If not get the balance
        Address.sendValue(payable(owner()), balance); //Send the balance to the owner
    }

    //This function returns the base URI
    //It can only be called from inside the contract
    function _baseURI() internal view override returns (string memory) {
        return baseURI;
    }

    //Function to set the refund address
    //This function can only be called from outside of the contract
    //Only owner can call this function
    function setRefundAddress(address _refundAddress) external onlyOwner {
        refundAddress = _refundAddress;
    }

    //Function to set the merkle root
    //This function can only be called from outside of the contract
    //Only owner can call this function
    function setMerkleRoot(bytes32 _root) external onlyOwner {
        merkleRoot = _root;
    }

    //Function to set the base uri
    //This function can only be called from outside of the contract
    //Only owner can call this function
    function setBaseURI(string memory uri) external onlyOwner {
        baseURI = uri;
    }

    //Function to set the refund time
    //Function can be called from anywhere
    //Only owner of the contract can call this function
    function toggleRefundCountdown() public onlyOwner {
        refundEndTime = block.timestamp + refundPeriod; //Adding the given time to the current time of the block
    }

    //Changes the presale status of the contract
    //This function can only be called from outside of the contract
    //Only owner can call this function
    function togglePresaleStatus() external onlyOwner {
        presaleActive = !presaleActive;
    }

    //Changes the public state status of the contract
    //This function can only be called from outside of the contract
    //Only owner can call this function
    function togglePublicSaleStatus() external onlyOwner {
        publicSaleActive = !publicSaleActive;
    }

    //Hashes the address and make it as a leaf to be used in the merkle tree
    //It can only be called from inside the function
    function _leaf(address _account) internal pure returns (bytes32) {
        return keccak256(abi.encodePacked(_account));
    }

    //Is the list in the allowed list
    //It can only be called from the inside of the contract
    function _isAllowlisted(
        address _account,
        bytes32[] calldata _proof,
        bytes32 _root
    ) internal pure returns (bool) {
        return MerkleProof.verify(_proof, _root, _leaf(_account));
    }
}

```
