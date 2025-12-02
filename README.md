/ SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/token/ERC20/extensions/ERC20Burnable.sol";
import "@openzeppelin/contracts/token/ERC20/extensions/ERC20Pausable.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";
import "@openzeppelin/contracts/token/ERC721/IERC721.sol";

contract MINIMON is ERC20, ERC20Burnable, ERC20Pausable, Ownable, ReentrancyGuard {
    // Максимальна емісія (наприклад 100 млн MINI)
    uint256 public constant MAX_SUPPLY = 100_000_000 * 10 ** 18;

    // Реферальна нагорода (в базисних пунктах – 10000 = 100%)
    uint256 public constant REFERRAL_BP = 300; // 3%
    uint256 public constant REF_DIVISOR = 10_000;

    // NFT, який дає право на бонус
    IERC721 public accessNFT;
    mapping(address => bool) public nftBonusClaimed;

    constructor() ERC20("MINIMON", "MINI") Ownable(msg.sender) {
        // Початковий мінт на твій гаманець (можеш змінити суму)
        _mint(msg.sender, 1_000_000 * 10 ** decimals());
    }

    // --- ПАУЗА ---

    function pause() external onlyOwner {
        _pause();
    }

    function unpause() external onlyOwner {
        _unpause();
    }

    // --- МІНТ З МАКС. ЕМІСІЄЮ ---

    function mint(address to, uint256 amount) external onlyOwner {
        require(totalSupply() + amount <= MAX_SUPPLY, "MAX_SUPPLY reached");
        _mint(to, amount);
    }

    // --- AIRDROP НА СПИСОК АДРЕС ---

    function airdrop(
        address[] calldata recipients,
        uint256[] calldata amounts
    ) external onlyOwner nonReentrant {
        require(recipients.length == amounts.length, "LEN_MISMATCH");

        for (uint256 i = 0; i < recipients.length; i++) {
            _transfer(msg.sender, recipients[i], amounts[i]);
        }
    }

    // --- AIRDROP + РЕФЕРАЛЬНА НАГОРОДА ---

    function airdropWithReferral(
        address to,
        uint256 amount,
        address referrer
    ) external onlyOwner nonReentrant {
        require(to != address(0), "ZERO_TO");
        uint256 remaining = amount;

        // Якщо є валідний реферал – нараховуємо % від суми
        if (referrer != address(0) && referrer != to) {
            uint256 refReward = (amount * REFERRAL_BP) / REF_DIVISOR;
            _transfer(msg.sender, referrer, refReward);
        }

        // Основна сума йде отримувачу
        _transfer(msg.sender, to, remaining);
    }

    // --- NFT UTILITY: БОНУС ДЛЯ ВЛАСНИКІВ NFT ---

    function setAccessNFT(address nft) external onlyOwner {
        accessNFT = IERC721(nft);
    }

    /// Користувач, який володіє певним NFT tokenId, може один раз забрати бонус
    /// Бонус відправляється з балансу owner() – слід мати запас MINI на гаманці власника
    function claimNftHolderBonus(
        uint256 tokenId,
        uint256 amount
    ) external nonReentrant {
        require(address(accessNFT) != address(0), "NFT_NOT_SET");
        require(accessNFT.ownerOf(tokenId) == msg.sender, "NOT_NFT_OWNER");
        require(!nftBonusClaimed[msg.sender], "ALREADY_CLAIMED");

        nftBonusClaimed[msg.sender] = true;
        _transfer(owner(), msg.sender, amount);
    }

    // --- Потрібний override для ERC20Pausable (OpenZeppelin v5) ---

    function _update(
        address from,
        address to,
        uint256 value
    ) internal override(ERC20, ERC20Pausable) {
        super._update(from, to, value);
    }
}
