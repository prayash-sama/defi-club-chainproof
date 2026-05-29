# defi-club-chainproof

### Smart Contract Code

<img width="1917" height="1078" alt="Screenshot 2026-05-29 222152" src="https://github.com/user-attachments/assets/cb0a17d0-670a-4c8c-883a-dbcd6c926e07" />

<img width="1903" height="942" alt="Screenshot 2026-05-29 221727" src="https://github.com/user-attachments/assets/3466845c-e80b-4537-9b35-b6fa58d8dc57" />



```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

contract Chainproof {
    address public admin;

    // Keeping this struct super lightweight to save gas.
    // We don't want to store huge github descriptions on-chain.
    struct Contributor {
        uint256 joinDate;
        uint256 reputationScore;
        bool isRegistered;
    }

    mapping(address => Contributor) public contributors;
    
    // whitelist of trusted people who can actually give out points
    // (this stops the sybil attack where someone makes 100 wallets to vouch for themselves)
    mapping(address => bool) public isReviewer;

    event ProfileRegistered(address indexed user, uint256 timestamp);
    event ContributionVerified(address indexed user, address indexed reviewer, string proofHash, uint256 pointsGranted);
    event ReviewerAdded(address indexed reviewer);

    modifier onlyAdmin() {
        require(msg.sender == admin, "Chainproof: Not admin");
        _;
    }

    modifier onlyReviewer() {
        require(isReviewer[msg.sender], "Chainproof: Not an authorized reviewer");
        _;
    }

    constructor() {
        admin = msg.sender;
        // deployer is automatically the first reviewer
        isReviewer[msg.sender] = true; 
    }

    function addReviewer(address _reviewer) public onlyAdmin {
        isReviewer[_reviewer] = true;
        emit ReviewerAdded(_reviewer);
    }

    // Users call this once to permanently lock in their join date.
    // This solves the cold start/veteran problem (Elara vs Magnus track requirement)
    function registerProfile() public {
        require(!contributors[msg.sender].isRegistered, "Chainproof: Already registered");

        contributors[msg.sender] = Contributor({
            joinDate: block.timestamp,
            reputationScore: 0,
            isRegistered: true
        });

        emit ProfileRegistered(msg.sender, block.timestamp);
    }

    function verifyWork(address _user, string memory _proofHash, uint256 _basePoints) public onlyReviewer {
        require(contributors[_user].isRegistered, "Chainproof: User needs to register first");

        // figure out how long they've been active in the community
        uint256 activeTime = block.timestamp - contributors[_user].joinDate;

        uint256 multiplier = 1;
        
        // if they've been around for over a year, give them a 2x bonus
        if (activeTime >= 365 days) {
            multiplier = 2;
        }

        uint256 finalPoints = _basePoints * multiplier;
        contributors[_user].reputationScore += finalPoints;

        // just emit the github hash instead of saving it to state to minimize gas costs
        emit ContributionVerified(_user, msg.sender, _proofHash, finalPoints);
    }

    function getReputation(address _user) public view returns (uint256) {
        return contributors[_user].reputationScore;
    }
}
