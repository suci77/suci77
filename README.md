### Hi there 👋

<!--
**suci77/suci77** is a ✨ _special_ ✨ repository because its `README.md` (this file) appears on your GitHub profile.

Here are some ideas to get you started:

- 🔭 I’m currently working on ...
- 🌱 I’m currently learning ...
- 👯 I’m looking to collaborate on ...
- 🤔 I’m looking for help with ...
- 💬 Ask me about ...
- 📫 How to reach me: ...
- 😄 Pronouns: ...
- ⚡ Fun fact: ...
-->


// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import {ERC20Capped} from "@openzeppelin/contracts/token/ERC20/extensions/ERC20Capped.sol";
import {ERC20Pausable} from "@openzeppelin/contracts/token/ERC20/extensions/ERC20Pausable.sol";
import {EIP712} from "@openzeppelin/contracts/utils/cryptography/EIP712.sol";
import {ECDSA} from "@openzeppelin/contracts/utils/cryptography/ECDSA.sol";
import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";
import {Ownable2Step} from "@openzeppelin/contracts/access/Ownable2Step.sol";

/// @notice ERC-20 reward token with capped supply and backend-authorized claims.
/// @dev Test thoroughly and obtain an independent audit before mainnet deployment.
contract DramaReward is ERC20, ERC20Capped, ERC20Pausable, EIP712, Ownable2Step {
    bytes32 private constant REWARD_TYPEHASH = keccak256(
        "Reward(address recipient,uint256 amount,bytes32 episodeId,uint256 nonce,uint256 deadline)"
    );

    address public rewardSigner;
    mapping(address account => uint256 nonce) public nonces;

    error ZeroAddress();
    error ZeroAmount();
    error SignatureExpired();
    error InvalidRewardSignature();

    event RewardSignerUpdated(address indexed previousSigner, address indexed newSigner);
    event RewardClaimed(
        address indexed recipient,
        uint256 amount,
        bytes32 indexed episodeId,
        uint256 nonce
    );

    /// @param initialOwner Gunakan multisig untuk administrasi kontrak.
    /// @param initialRewardSigner Alamat signer backend; private key harus tetap di server.
    /// @param maxSupply Batas suplai dalam base units (token memakai 18 desimal).
    constructor(address initialOwner, address initialRewardSigner, uint256 maxSupply)
        ERC20("Drama Reward", "DRM")
        ERC20Capped(maxSupply)
        EIP712("Drama Reward", "1")
        Ownable(initialOwner)
    {
        if (initialOwner == address(0) || initialRewardSigner == address(0)) {
            revert ZeroAddress();
        }

        rewardSigner = initialRewardSigner;
        emit RewardSignerUpdated(address(0), initialRewardSigner);
    }

    /// @notice Klaim reward yang telah disetujui signer backend.
    /// @param amount Jumlah reward dalam base units.
    /// @param episodeId ID episode berupa bytes32.
    /// @param deadline Batas waktu klaim dalam Unix timestamp.
    /// @param signature Tanda tangan EIP-712 backend untuk wallet dan nonce ini.
    function claimReward(
        uint256 amount,
        bytes32 episodeId,
        uint256 deadline,
        bytes calldata signature
    ) external whenNotPaused {
        if (amount == 0) revert ZeroAmount();
        if (block.timestamp > deadline) revert SignatureExpired();

        uint256 nonce = nonces[msg.sender];
        bytes32 structHash(
            abi.encode(
                REWARD_TYPEHASH,
                msg.sender,
                amount,
                episodeId,
                nonce,
                deadline
            )
        );

        address recovered = ECDSA.recover(_hashTypedDataV4(structHash), signature);
        if (recovered != rewardSigner) revert InvalidRewardSignature        _mint(msg.sender, amount); // ERC20Capped menegakkan batas suplai.
        emit RewardClaimed(msg.sender, amount, episodeId, nonce);
    }

    function setRewardSigner(address newSigner) external onlyOwner {
        if (newSigner == address(0)) revert ZeroAddress();

        address previousSigner = rewardSigner;
        rewardSigner = newSigner;
        emit RewardSignerUpdated(previousSigner, newSigner);
    }

    function pause() external onlyOwner {
        _pause();
    }

    function unpause() external onlyOwner {
        _unpause();
    }

    function _update(address from, address to, uint256 value)
        internal
        override(ERC20, ERC20Capped, ERC20Pausable)
    {
        super._, value);
    }
}
