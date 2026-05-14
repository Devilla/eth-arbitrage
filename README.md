// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

interface IERC20 {
    function balanceOf(address account) external view returns (uint256);
    function allowance(address owner, address spender) external view returns (uint256);
    function approve(address spender, uint256 amount) external returns (bool);
    function transfer(address to, uint256 amount) external returns (bool);
}

interface IUniswapV2Router {
    function getAmountsOut(
        uint256 amountIn,
        address[] calldata path
    ) external view returns (uint256[] memory amounts);

    function swapExactTokensForTokens(
        uint256 amountIn,
        uint256 amountOutMin,
        address[] calldata path,
        address to,
        uint256 deadline
    ) external returns (uint256[] memory amounts);
}

contract EthArbitrage {
    address public owner;

    error NotOwner();
    error BadSlippage();
    error BadPath();
    error NotProfitable();
    error TransferFailed();
    error ApprovalFailed();

    modifier onlyOwner() {
        if (msg.sender != owner) revert NotOwner();
        _;
    }

    constructor() {
        owner = msg.sender;
    }

    function executeArbitrage(
        address routerA,
        address routerB,
        address[] calldata pathA,
        address[] calldata pathB,
        uint256 amountIn,
        uint256 slippageBps,
        uint256 minProfit
    ) external onlyOwner {
        if (pathA.length < 2 || pathB.length < 2) revert BadPath();
        if (pathA[0] != pathB[pathB.length - 1]) revert BadPath();
        if (pathA[pathA.length - 1] != pathB[0]) revert BadPath();

        address baseToken = pathA[0];
        address intermediateToken = pathA[pathA.length - 1];

        uint256 initialBalance = IERC20(baseToken).balanceOf(address(this));

        _approveIfNeeded(baseToken, routerA, amountIn);

        uint256[] memory quoteA = IUniswapV2Router(routerA).getAmountsOut(amountIn, pathA);
        uint256 minOutA = _minOut(quoteA[quoteA.length - 1], slippageBps);

        uint256[] memory swapA = IUniswapV2Router(routerA).swapExactTokensForTokens(
            amountIn,
            minOutA,
            pathA,
            address(this),
            block.timestamp
        );

        uint256 intermediateAmount = swapA[swapA.length - 1];

        _approveIfNeeded(intermediateToken, routerB, intermediateAmount);

        uint256[] memory quoteB = IUniswapV2Router(routerB).getAmountsOut(intermediateAmount, pathB);
        uint256 minOutB = _minOut(quoteB[quoteB.length - 1], slippageBps);

        IUniswapV2Router(routerB).swapExactTokensForTokens(
            intermediateAmount,
            minOutB,
            pathB,
            address(this),
            block.timestamp
        );

        uint256 finalBalance = IERC20(baseToken).balanceOf(address(this));

        if (finalBalance <= initialBalance + minProfit) revert NotProfitable();
    }

    function quoteArbitrage(
        address routerA,
        address routerB,
        address[] calldata pathA,
        address[] calldata pathB,
        uint256 amountIn,
        uint256 slippageBps
    )
        external
        view
        returns (
            uint256 quotedOutA,
            uint256 minOutA,
            uint256 quotedOutB,
            uint256 minOutB
        )
    {
        if (pathA.length < 2 || pathB.length < 2) revert BadPath();
        if (pathA[0] != pathB[pathB.length - 1]) revert BadPath();
        if (pathA[pathA.length - 1] != pathB[0]) revert BadPath();

        uint256[] memory quoteA = IUniswapV2Router(routerA).getAmountsOut(amountIn, pathA);
        quotedOutA = quoteA[quoteA.length - 1];
        minOutA = _minOut(quotedOutA, slippageBps);

        uint256[] memory quoteB = IUniswapV2Router(routerB).getAmountsOut(minOutA, pathB);
        quotedOutB = quoteB[quoteB.length - 1];
        minOutB = _minOut(quotedOutB, slippageBps);
    }

    function withdrawToken(address token, uint256 amount, address to) external onlyOwner {
        bool ok = IERC20(token).transfer(to, amount);
        if (!ok) revert TransferFailed();
    }

    function transferOwnership(address newOwner) external onlyOwner {
        owner = newOwner;
    }

    function _minOut(uint256 quotedOut, uint256 slippageBps) internal pure returns (uint256) {
        if (slippageBps > 10_000) revert BadSlippage();
        return (quotedOut * (10_000 - slippageBps)) / 10_000;
    }

    function _approveIfNeeded(address token, address spender, uint256 amount) internal {
        uint256 currentAllowance = IERC20(token).allowance(address(this), spender);

        if (currentAllowance < amount) {
            bool resetOk = IERC20(token).approve(spender, 0);
            if (!resetOk) revert ApprovalFailed();

            bool approveOk = IERC20(token).approve(spender, amount);
            if (!approveOk) revert ApprovalFailed();
        }
    }
}