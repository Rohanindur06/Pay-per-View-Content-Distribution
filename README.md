// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract PayPerView {
    address public owner;

    struct Content {
        uint256 id;
        string name;
        string uri;
        uint256 price;
        uint256 views;
        uint256 totalRatings;
        uint256 ratingSum;
        address creator;
    }

    mapping(uint256 => Content) public contentMap;
    mapping(address => mapping(uint256 => bool)) public userAccess;

    event ContentCreated(uint256 id, address creator, string name, uint256 price);
    event ContentPurchased(address user, uint256 contentId, uint256 price);
    event ContentRated(address user, uint256 contentId, uint256 rating);
    event ContentRemoved(uint256 id, address creator);
    event ContentPriceUpdated(uint256 id, uint256 newPrice);

    constructor() {
        owner = msg.sender;
    }

    function addContent(string memory _name, string memory _uri, uint256 _price) public {
        uint256 contentId = uint256(keccak256(abi.encodePacked(_name, _uri, block.timestamp)));
        contentMap[contentId] = Content({
            id: contentId,
            name: _name,
            uri: _uri,
            price: _price,
            views: 0,
            totalRatings: 0,
            ratingSum: 0,
            creator: msg.sender
        });
        emit ContentCreated(contentId, msg.sender, _name, _price);
    }

    function purchaseContent(uint256 _contentId) public payable {
        Content storage content = contentMap[_contentId];
        require(content.creator != address(0), "Content does not exist");
        require(msg.value >= content.price, "Insufficient funds to purchase content");
        require(!userAccess[msg.sender][_contentId], "Content already purchased");

        content.views += 1;
        userAccess[msg.sender][_contentId] = true;
        payable(content.creator).transfer(msg.value);

        emit ContentPurchased(msg.sender, _contentId, content.price);
    }

    function rateContent(uint256 _contentId, uint256 _rating) public {
        require(_rating >= 1 && _rating <= 5, "Rating must be between 1 and 5");
        Content storage content = contentMap[_contentId];
        require(content.creator != address(0), "Content does not exist");

        content.totalRatings += 1;
        content.ratingSum += _rating;
        
        emit ContentRated(msg.sender, _contentId, _rating);
    }

    function getAverageRating(uint256 _contentId) public view returns (uint256) {
        Content storage content = contentMap[_contentId];
        if (content.totalRatings == 0) {
            return 0;
        }
        return content.ratingSum / content.totalRatings;
    }

    function getContentStats(uint256 _contentId) public view returns (uint256 views, uint256 averageRating) {
        Content storage content = contentMap[_contentId];
        views = content.views;
        averageRating = getAverageRating(_contentId);
    }

    // New Functions 👇

    // Function to remove content (only creator can remove)
    function removeContent(uint256 _contentId) public {
        Content storage content = contentMap[_contentId];
        require(content.creator == msg.sender, "Only creator can remove content");
        delete contentMap[_contentId];
        emit ContentRemoved(_contentId, msg.sender);
    }

    // Function to update content price (only creator can update)
    function updateContentPrice(uint256 _contentId, uint256 _newPrice) public {
        Content storage content = contentMap[_contentId];
        require(content.creator == msg.sender, "Only creator can update price");
        content.price = _newPrice;
        emit ContentPriceUpdated(_contentId, _newPrice);
    }

    // Function to check if a user has purchased specific content
    function hasAccess(address _user, uint256 _contentId) public view returns (bool) {
        return userAccess[_user][_contentId];
    }

    // Function to get full content details
    function getContentDetails(uint256 _contentId) public view returns (
        string memory name,
        string memory uri,
        uint256 price,
        uint256 views,
        uint256 totalRatings,
        uint256 ratingSum,
        address creator
    ) {
        Content storage content = contentMap[_contentId];
        name = content.name;
        uri = content.uri;
        price = content.price;
        views = content.views;
        totalRatings = content.totalRatings;
        ratingSum = content.ratingSum;
        creator = content.creator;
    }
}
