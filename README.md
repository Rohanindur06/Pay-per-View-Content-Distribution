// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract PayPerView {
    // Declare the contract owner (admin)
    address public owner;
    
    // Declare a structure for content
    struct Content {
        uint256 id;
        string name;
        string uri;  // URI to access the content (can be IPFS link)
        uint256 price;
        uint256 views;
        uint256 totalRatings;
        uint256 ratingSum;
        address creator;
        bool isActive; // Flag to track whether the content is active
    }

    // Mapping to store content by ID
    mapping(uint256 => Content) public contentMap;
    
    // Mapping to track payments made by users
    mapping(address => mapping(uint256 => bool)) public userAccess;

    // Event for content creation
    event ContentCreated(uint256 id, address creator, string name, uint256 price);

    // Event for a payment transaction
    event ContentPurchased(address user, uint256 contentId, uint256 price);

    // Event for content rating
    event ContentRated(address user, uint256 contentId, uint256 rating);

    // Event for refund request
    event RefundRequested(address user, uint256 contentId, uint256 price);

    // Constructor to set the contract owner
    constructor() {
        owner = msg.sender;
    }

    // Function to add new content to the platform
    function addContent(string memory _name, string memory _uri, uint256 _price) public {
        uint256 contentId = uint256(keccak256(abi.encodePacked(_name, _uri, block.timestamp))); // generate unique content ID
        contentMap[contentId] = Content({
            id: contentId,
            name: _name,
            uri: _uri,
            price: _price,
            views: 0,
            totalRatings: 0,
            ratingSum: 0,
            creator: msg.sender,
            isActive: true
        });
        emit ContentCreated(contentId, msg.sender, _name, _price);
    }

    // Function to purchase access to content
    function purchaseContent(uint256 _contentId) public payable {
        Content storage content = contentMap[_contentId];
        
        require(content.isActive, "Content is no longer available");
        require(msg.value >= content.price, "Insufficient funds to purchase content");
        require(!userAccess[msg.sender][_contentId], "Content already purchased");

        content.views += 1;
        userAccess[msg.sender][_contentId] = true;
        payable(content.creator).transfer(msg.value); // Transfer payment to the content creator

        emit ContentPurchased(msg.sender, _contentId, content.price);
    }

    // Function to rate content (1 to 5 scale)
    function rateContent(uint256 _contentId, uint256 _rating) public {
        require(_rating >= 1 && _rating <= 5, "Rating must be between 1 and 5");
        Content storage content = contentMap[_contentId];

        content.totalRatings += 1;
        content.ratingSum += _rating;
        
        emit ContentRated(msg.sender, _contentId, _rating);
    }

    // Function to get the average rating for a content piece
    function getAverageRating(uint256 _contentId) public view returns (uint256) {
        Content storage content = contentMap[_contentId];
        if (content.totalRatings == 0) {
            return 0;
        }
        return content.ratingSum / content.totalRatings;
    }

    // Function to get content statistics
    function getContentStats(uint256 _contentId) public view returns (uint256 views, uint256 averageRating) {
        Content storage content = contentMap[_contentId];
        return (content.views, getAverageRating(_contentId));
    }

    // Function to request a refund (if conditions are met)
    function requestRefund(uint256 _contentId) public {
        Content storage content = contentMap[_contentId];

        require(userAccess[msg.sender][_contentId], "You must have purchased the content to request a refund");
        require(!content.isActive, "Refunds are not available for active content");

        uint256 refundAmount = content.price;
        userAccess[msg.sender][_contentId] = false; // Revoke user access
        payable(msg.sender).transfer(refundAmount); // Refund the user
        emit RefundRequested(msg.sender, _contentId, refundAmount);
    }

    // Function to update content details (name, URI, or price)
    function updateContent(uint256 _contentId, string memory _newName, string memory _newUri, uint256 _newPrice) public {
        Content storage content = contentMap[_contentId];
        
        require(content.creator == msg.sender, "Only the content creator can update content");
        
        content.name = _newName;
        content.uri = _newUri;
        content.price = _newPrice;
    }

    // Function to deactivate content (e.g., remove from platform)
    function deactivateContent(uint256 _contentId) public {
        Content storage content = contentMap[_contentId];
        
        require(content.creator == msg.sender, "Only the content creator can deactivate content");

        content.isActive = false;
    }

    // Function for platform owner to remove content
    function removeContent(uint256 _contentId) public {
        require(msg.sender == owner, "Only the contract owner can remove content");

        delete contentMap[_contentId]; // Remove the content from the platform
    }

    // Function to withdraw contract funds (only owner)
    function withdrawFunds() public {
        require(msg.sender == owner, "Only the


