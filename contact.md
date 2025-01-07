// SPDX-License-Identifier: GPL-3.0
pragma solidity ^0.8.4;

interface AggregatorV3Interface {
    function decimals() external view returns (uint8);

    function latestRoundData()
    external
    view
    returns (
        uint80 roundId,
        int256 answer,
        uint256 startedAt,
        uint256 updatedAt,
        uint80 answeredInRound
    );
}

contract DecentraGuysToken
{

    string public constant name = "DecentraGuys";
    string public constant symbol = "DECE";
    uint8 public constant decimals = 8;

    address public bnbUsdPriceFeed = 0x0567F2323251f0Aab15c8dFb1967E4e8A7D42aeE;
    uint8 public usdtDecimals = 18;

    event Approval(address indexed tokenOwner, address indexed spender, uint tokens);
    event Transfer(address indexed from, address indexed to, uint tokens);
    event ITOSell(address indexed user, uint indexed itoLevel, uint amount);
    event Airdrop(address indexed user, uint amount);
    event GameBoxTransfer(address indexed user, uint amount);
    event GameBoxCharged(uint amount);

    mapping(address => uint256) private balances;
    mapping(address => mapping(address => uint256)) private allowed;
    uint256 public totalSupply = 210000000000000000;

    //authentication
    address private ownerAddress;
    mapping(address => bool) private authorizedWallets;
    address[] private authorizedAddresses;

    uint public devPercent = 10;
    uint public airdropPercent = 5;
    uint public itoPercent = 30;
    uint public gameBoxPercent = 55;
    uint public totalItoAmount;
    uint public airdropRemaining;
    uint public gameBoxRemaining;

    uint public affiliateCommission;
    mapping(address => address) private referrer;
    mapping(address => uint) private referrerIncome;
    mapping(address => address[]) private referrals;
    mapping(address => uint) private specialCommission;

    mapping(address => bool) private stakeWalletExists;
    mapping(address => uint) private stakeShare;
    address[] private stakeAddresses;

    uint[] private itoPercents = [25, 25, 20, 20, 10];
    uint[] private itoRates = [250, 200, 150, 100, 50]; // how many tokens for 1 usd
    uint[] private itoRemainingAmounts;
    uint private itoCurrentLevel = 0;
    bool public itoIsActive = false;

    struct Chest
    {
        uint id;
        uint tokens; // amount of tokens with decimals included
        uint price; //how much usdt for this chest with decimals included
        uint remaining; //
        uint sold;
        bool isActive;
    }

    struct UserChest
    {
        uint id;
        address user;
        uint chestId;
    }

    Chest[] private chests;
    address[] private soldChestUsers;
    uint[] private soldChestIds;

    using SafeMath for uint256;

    AggregatorV3Interface internal priceFeed;

    constructor()
    {
        balances[address(this)] = totalSupply;
        ownerAddress = msg.sender;

        priceFeed = AggregatorV3Interface(bnbUsdPriceFeed);

        authorizedWallets[msg.sender] = true;
        authorizedAddresses.push(msg.sender);

        totalItoAmount = totalSupply * itoPercent / 100;
        airdropRemaining = totalSupply * airdropPercent / 100;
        gameBoxRemaining = totalSupply * gameBoxPercent / 100;

        for (uint i = 0; i < itoPercents.length; i++)
        {
            itoRemainingAmounts.push(totalItoAmount * itoPercents[i] / 100);
        }
    }

    function balanceOf(address tokenOwner) external view returns (uint256) {
        return balances[tokenOwner];
    }

    function transfer(address receiver, uint256 _amount) external returns (bool)
    {
        require(_amount <= balances[msg.sender], "transfer error");
        _transfer(msg.sender, receiver, _amount);
        return true;
    }

    function approve(address delegate, uint256 numTokens) external returns (bool) {
        allowed[msg.sender][delegate] = numTokens;
        emit Approval(msg.sender, delegate, numTokens);
        return true;
    }

    function allowance(address owner, address delegate) external view returns (uint) {
        return allowed[owner][delegate];
    }

    function transferFrom(address _owner, address _receiver, uint256 _amount) external returns (bool) {
        require(_amount <= balances[_owner]);
        require(_amount <= allowed[_owner][msg.sender]);

        _transfer(_owner, _receiver, _amount);
        return true;
    }

    //------------- Admin Methods -------------//

    function setStakeWallets(address[] memory wallets, uint[] memory shares) external
    {
        require(stakeAddresses.length == 0, 'already set');
        require(wallets.length == shares.length, 'invalid input');

        uint totalPercent = 0;
        for (uint i = 0; i < wallets.length; i++)
        {
            totalPercent += shares[i];
        }
        require(totalPercent == 100, 'invalid shares');
        uint devShare = totalSupply * devPercent / 100;


        for (uint i = 0; i < wallets.length; i++)
        {
            stakeWalletExists[wallets[i]] = true;
            stakeAddresses.push(wallets[i]);
            stakeShare[wallets[i]] = shares[i];

            _transfer(address(this), wallets[i], (devShare * shares[i] / 100));
        }
    }

    function withdraw(uint256 _amount) external onlyAuthorized
    {
        require(address(this).balance >= _amount, "insufficient balance");
        require(stakeAddresses.length > 0, "no stake wallet");

        for (uint i = 0; i < stakeAddresses.length; i++)
        {
            address payable wallet = payable(stakeAddresses[i]);
            wallet.transfer(_amount.mul(stakeShare[stakeAddresses[i]]).div(100));
        }
    }

    function addChest(uint _tokens, uint _price, uint _totalForSale, bool _isActive) external onlyAuthorized
    {
        chests.push(Chest({
            id: (chests.length + 1),
            price: _price,
            tokens: _tokens,
            remaining: _totalForSale,
            isActive: _isActive,
            sold: 0
        }));
    }

    function editChest(uint _id, uint _tokens, uint _price, uint _totalForSale, bool _isActive) external onlyAuthorized
    {
        require(_id <= chests.length, 'invalid id');

        uint _i = _id - 1;
        chests[_i].tokens = _tokens;
        chests[_i].price = _price;
        chests[_i].remaining = _totalForSale - chests[_i].sold;
        chests[_i].isActive = _isActive;
    }

    function setPriceFeed(address _feedAddress,uint8 _usdtDecimals) external onlyAuthorized
    {
        bnbUsdPriceFeed = _feedAddress;
        priceFeed = AggregatorV3Interface(bnbUsdPriceFeed);
        usdtDecimals = _usdtDecimals;
    }

    function setItoRates(uint[] memory _rates) external onlyAuthorized
    {
        require(_rates.length == itoPercents.length, 'invalid rates');
        itoRates = _rates;
    }

    function activateIto() external onlyAuthorized
    {
        require(!itoIsActive, 'ITO is already active');
        require(itoCurrentLevel < itoPercents.length, 'ITO levels finished');

        itoCurrentLevel = itoCurrentLevel + 1;
        itoIsActive = true;
    }

    function deactivateIto() external onlyAuthorized
    {
        require(itoIsActive, 'ITO is already inactive');

        itoIsActive = false;

        // burn remaining amount
        _transfer(address(this), address(0), itoRemainingAmounts[itoCurrentLevel]);
    }

    function transferAirdrop(address _targetAddress, uint _amount) external onlyAuthorized
    {
        require(airdropRemaining >= _amount, 'insufficient airdrop balance');

        _transfer(address(this), _targetAddress, _amount);

        airdropRemaining = airdropRemaining.sub(_amount);

        emit Airdrop(_targetAddress, _amount);
    }

    function transferFromGameBox(address _targetAddress, uint _amount) external onlyAuthorized
    {
        require(gameBoxRemaining >= _amount, 'insufficient game box balance');

        _transfer(address(this), _targetAddress, _amount);

        gameBoxRemaining = gameBoxRemaining.sub(_amount);

        emit GameBoxTransfer(_targetAddress, _amount);
    }

    function chargeGameBox(uint _amount) external onlyAuthorized
    {
        _transfer(msg.sender, address(this), _amount);

        gameBoxRemaining = gameBoxRemaining.add(_amount);

        emit GameBoxCharged(_amount);
    }

    function setAffiliateCommission(uint _a) external onlyAuthorized
    {
        require(_a < 31, 'invalid percent');
        affiliateCommission = _a;
    }

    function setSpecialCommission(address _ad, uint _a) external onlyAuthorized
    {
        require(_a < 31, 'invalid percent');
        specialCommission[_ad] = _a;
    }

    function getStakeAddresses() external view onlyAuthorized returns (address[] memory)
    {
        return stakeAddresses;
    }

    function getStakeAddressShare(address _ad) external view onlyAuthorized returns (uint)
    {
        return stakeShare[_ad];
    }

    function getSpecialCommission(address _ad) external view onlyAuthorized returns (uint)
    {
        return specialCommission[_ad];
    }

    //------------- DecentraGuys Methods -------------//

    function getItoDetails() external view returns (bool, uint, uint[] memory, uint[] memory, uint[] memory)
    {
        return (itoIsActive, itoCurrentLevel, itoPercents, itoRates, itoRemainingAmounts);
    }

    function isAuthorized(address _ad) external view returns (bool)
    {
        return authorizedWallets[_ad];
    }

    function getReferrer(address _ad) external view returns (address)
    {
        return referrer[_ad];
    }

    receive() payable external
    {
        buy();
    }

    function buyWithReferrer(address _referrer) external payable
    {
        if (referrer[msg.sender] == address(0) && msg.sender != _referrer)
        {
            referrer[msg.sender] = _referrer;
            referrals[_referrer].push(msg.sender);
        }
        buy();
    }

    function buy() public payable
    {
        require(itoIsActive, 'ITO is not active');

        uint _index = (itoCurrentLevel - 1);
        uint256 _amountToBuy = convertMsgValueToUSDT() * itoRates[_index] * (10 ** decimals) / (10 ** usdtDecimals);

        require(_amountToBuy <= itoRemainingAmounts[_index], "not enough token in reserve");

        if (referrer[msg.sender] != address(0))
        {
            uint _p = affiliateCommission;
            if (specialCommission[referrer[msg.sender]] != 0)
                _p = specialCommission[referrer[msg.sender]];
            uint _a = msg.value.mul(_p).div(100);

            address payable _wallet = payable(referrer[msg.sender]);
            _wallet.transfer(_a);
            referrerIncome[referrer[msg.sender]] += _a;
        }

        _transfer(address(this), msg.sender, _amountToBuy);
        itoRemainingAmounts[_index] -= _amountToBuy;
        emit ITOSell(msg.sender, itoCurrentLevel, _amountToBuy);
    }

    function canBuyChest(uint _chestId) external view returns (uint)
    {
        if (!itoIsActive)
            return 1;
        if (_chestId > chests.length)
            return 2;
        if (!chests[_chestId - 1].isActive)
            return 3;
        if (chests[_chestId - 1].remaining <= 0)
            return 4;
        return 0;
    }

    function buyChest(uint _chestId, address _referrer) public payable
    {
        require(itoIsActive, 'ITO is not active');
        require(_chestId <= chests.length, 'invalid id');
        require(chests[_chestId - 1].isActive, 'chest is not active');
        require(chests[_chestId - 1].remaining > 0, 'sold out');
        require(convertMsgValueToUSDT() >= chests[_chestId - 1].price, 'invalid payment');

        if (referrer[msg.sender] == address(0) && msg.sender != _referrer)
        {
            referrer[msg.sender] = _referrer;
            referrals[_referrer].push(msg.sender);
        }

        if (referrer[msg.sender] != address(0))
        {
            uint _p = affiliateCommission;
            if (specialCommission[referrer[msg.sender]] != 0)
                _p = specialCommission[referrer[msg.sender]];
            uint _a = msg.value.mul(_p).div(100);

            address payable _wallet = payable(referrer[msg.sender]);
            _wallet.transfer(_a);

            referrerIncome[referrer[msg.sender]] += _a;
        }
        _transfer(address(this), msg.sender, chests[_chestId - 1].tokens);
        chests[_chestId - 1].remaining -= 1;
        chests[_chestId - 1].sold += 1;

        soldChestUsers.push(msg.sender);
        soldChestIds.push(_chestId);
    }

    function getChests() external view returns (Chest[] memory)
    {
        return chests;
    }

    function getUserChestsLength() external view returns (uint)
    {
        return soldChestUsers.length;
    }

    function getUserChests(uint _of, uint _n, int8 _sort) external view returns (UserChest[] memory)
    {
        if (_of > (soldChestUsers.length - 1))
            return new UserChest[](0);
        if ((_of + _n) > (soldChestUsers.length - 1))
            _n = soldChestUsers.length - _of;

        UserChest[] memory o = new UserChest[](_n);
        uint256 index = 0;
        for (uint i = _of; index < _n; i++)
        {
            uint t = i;
            if (_sort == - 1)
                t = soldChestUsers.length - i - 1;
            o[index].id = (i + 1);
            o[index].user = soldChestUsers[t];
            o[index].chestId = soldChestIds[t];
            index++;
        }
        return o;
    }

    //-------------------------------------------------//

    /**
     * @notice Gets the latest BNB price in USD (with Chainlink decimals)
     * @return bnbPrice The price of 1 BNB in USD
     */
    function getBnbPriceInUSD() internal view returns (uint256) {
        (, int256 price, , ,) = priceFeed.latestRoundData();
        require(price > 0, "Invalid price");
        return uint256(price); // Price has 8 decimals
    }

    /**
     * @notice Converts msg.value (BNB sent to the contract) to its USDT equivalent
     * @return usdtAmount The equivalent amount of USDT (6 decimals)
     */
    function convertMsgValueToUSDT() internal view returns (uint256)
    {
        uint256 bnbAmount = msg.value; // BNB amount in wei (18 decimals)

        // Get BNB/USD price from Chainlink
        uint256 bnbPrice = getBnbPriceInUSD(); // Price has 8 decimals

        // Chainlink price decimals (8) and USDT decimals (6)
        uint8 priceDecimals = priceFeed.decimals(); // Typically 8

        // Normalize BNB amount to USD in smallest USDT units (6 decimals)
        uint256 usdtAmount = (bnbAmount * bnbPrice) /
            (10 ** priceDecimals); // Normalize to 18 decimals

        return usdtAmount / (10 ** (18 - usdtDecimals)); // Normalize to USDT decimals
    }

    /**
     * @notice Converts msg.value (BNB sent to the contract) to its USDT equivalent
     * @return usdtAmount The equivalent amount of USDT (6 decimals)
     */
    function convertMsgValueToUSDTTest(uint256 bnbAmount) public view returns (uint256)
    {
        // Get BNB/USD price from Chainlink
        uint256 bnbPrice = getBnbPriceInUSD(); // Price has 8 decimals

        // Chainlink price decimals (8) and USDT decimals (6)
        uint8 priceDecimals = priceFeed.decimals(); // Typically 8

        // Normalize BNB amount to USD in smallest USDT units (6 decimals)
        uint256 usdtAmount = (bnbAmount * bnbPrice) /
            (10 ** priceDecimals); // Normalize to 18 decimals

        return usdtAmount / (10 ** (18 - usdtDecimals)); // Normalize to USDT decimals
    }

    function _transfer(address _sender, address _receiver, uint256 _amount) private
    {
        require(_amount <= balances[_sender], "insufficient balance");

        balances[_sender] = balances[_sender].sub(_amount);
        balances[_receiver] = balances[_receiver].add(_amount);
        emit Transfer(_sender, _receiver, _amount);

        if (_receiver == address(0))
        {
            totalSupply = totalSupply.sub(_amount);
        }
    }

    modifier onlyAuthorized()
    {
        require(authorizedWallets[msg.sender], "you are not authorized for this operation");
        _;
    }
}

library SafeMath
{

    function tryAdd(uint256 a, uint256 b) internal pure returns (bool, uint256)
    {
        unchecked {
            uint256 c = a + b;
            if (c < a) return (false, 0);
            return (true, c);
        }
    }

    function trySub(uint256 a, uint256 b) internal pure returns (bool, uint256) {
        unchecked {
            if (b > a) return (false, 0);
            return (true, a - b);
        }
    }

    function tryMul(uint256 a, uint256 b) internal pure returns (bool, uint256) {
        unchecked {
        // Gas optimization: this is cheaper than requiring 'a' not being zero, but the
        // benefit is lost if 'b' is also tested.
        // See: https://github.com/OpenZeppelin/openzeppelin-contracts/pull/522
            if (a == 0) return (true, 0);
            uint256 c = a * b;
            if (c / a != b) return (false, 0);
            return (true, c);
        }
    }

    function tryDiv(uint256 a, uint256 b) internal pure returns (bool, uint256) {
        unchecked {
            if (b == 0) return (false, 0);
            return (true, a / b);
        }
    }

    function tryMod(uint256 a, uint256 b) internal pure returns (bool, uint256) {
        unchecked {
            if (b == 0) return (false, 0);
            return (true, a % b);
        }
    }

    function add(uint256 a, uint256 b) internal pure returns (uint256) {
        return a + b;
    }

    function sub(uint256 a, uint256 b) internal pure returns (uint256) {
        return a - b;
    }

    function mul(uint256 a, uint256 b) internal pure returns (uint256) {
        return a * b;
    }

    function div(uint256 a, uint256 b) internal pure returns (uint256) {
        return a / b;
    }

    function mod(uint256 a, uint256 b) internal pure returns (uint256) {
        return a % b;
    }

    function sub(
        uint256 a,
        uint256 b,
        string memory errorMessage
    ) internal pure returns (uint256) {
        unchecked {
            require(b <= a, errorMessage);
            return a - b;
        }
    }

    function div(
        uint256 a,
        uint256 b,
        string memory errorMessage
    ) internal pure returns (uint256) {
        unchecked {
            require(b > 0, errorMessage);
            return a / b;
        }
    }

    function mod(
        uint256 a,
        uint256 b,
        string memory errorMessage
    ) internal pure returns (uint256) {
        unchecked {
            require(b > 0, errorMessage);
            return a % b;
        }
    }
}
