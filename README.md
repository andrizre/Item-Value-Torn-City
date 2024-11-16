// ==UserScript==
// @name         Item Page Total Value
// @namespace    https://arsonwarehouse.com
// @version      1.1
// @description  Shows the market value of each item stack on your items page.
// @author       Sulsay [2173590]
// @match        https://www.torn.com/item.php
// @grant        none
// @run-at       document-start
// @downloadURL https://update.greasyfork.org/scripts/387799/Item%20Page%20Total%20Value.user.js
// @updateURL https://update.greasyfork.org/scripts/387799/Item%20Page%20Total%20Value.meta.js
// ==/UserScript==

const settings = {
    tornApiKey: '6Tptwn7osFeqXFVp',
};

const marketValueByItemId = new Map();

(function() {
    onLoadItems(insertValueIntoItemsOfVisibleCategory);
})();

async function insertValueIntoItemsOfVisibleCategory() {
    const itemListItems = Array.from(getVisibleCategory().children).filter(li => li.hasAttribute('data-item'));
    const itemIds = itemListItems.map(item => parseInt(item.dataset.item, 10));

    try {
        await ensureMarketValueIsLoadedForItems(itemIds);
        itemListItems.forEach(insertValueIntoItemRow);
    } catch (error) {
        alert('The user script "Item Page Total Value" could not load items data. Did you enter your API key?\n\nTorn API says: ' + error.message);
    }
}

function insertValueIntoItemRow(itemListItem) {
    if (itemListItem.dataset.valueIsInserted === 'true') {
        return;
    }

    const itemId = parseInt(itemListItem.dataset.item, 10);
    const nameWrapSpan = itemListItem.querySelector('.name-wrap');
    const firstQuantitySpan = nameWrapSpan.querySelector('.qty');
    const quantity = parseInt(firstQuantitySpan.innerText.trim().replace('x', ''), 10) || 1;

    const marketValue = marketValueByItemId.get(itemId);
    const totalValue = quantity * marketValue;

    let displayValue;

    if (quantity === 1) {
        // Jika hanya 1 item, tampilkan harga satuannya
        displayValue = `${formatCurrency(marketValue)}`;
    } else {
        // Jika lebih dari 1, tampilkan format lengkap dengan warna putih pada bagian hasil kali dan warna hijau pada total harga
        displayValue = `
            ${formatCurrency(marketValue)} | 
            <span style="color:#FFFFFF;">${quantity}×</span> <span style="color:#FFFFFF;">=</span> <span style="color:#7ec11e;">${formatCurrency(totalValue)}</span>
        `;
    }

    nameWrapSpan.insertAdjacentHTML('beforeend', `
        <span style="position:absolute;right:.5rem;color:#7ec11e">
            ${displayValue}
        </span>
    `);
    itemListItem.dataset.valueIsInserted = 'true';
}
function ensureMarketValueIsLoadedForItems(itemIds) {
    const itemIdsToFetch = [];
    for (let itemId of itemIds) {
        if (! marketValueByItemId.has(itemId)) {
            itemIdsToFetch.push(itemId);
        }
    }

    if (itemIdsToFetch.length === 0) {
        return Promise.resolve();
    }

    return fetch('https://api.torn.com/torn/' + itemIdsToFetch.join(',') + '?selections=items&key=' + settings.tornApiKey)
        .then(response => response.json())
        .then(response => {
            if (response.items === undefined) {
                throw Error((response.error && response.error.error) || 'Unknown error');
            }
            return response.items;
        })
        .then(populateMarketValueByItemId);
}

function populateMarketValueByItemId(items) {
    Object.entries(items).forEach(([itemId, apiData]) => {
        marketValueByItemId.set(parseInt(itemId, 10), apiData.market_value);
    });
}

function onLoadItems(onLoadHandler) {
    const xhrOpen = window.XMLHttpRequest.prototype.open;
    window.XMLHttpRequest.prototype.open = function (method, url) {
        if (method.toUpperCase() === 'POST' && url.substr(0, 14) === 'item.php?rfcv=') {
            this.onload = onLoadHandler;
        }
        xhrOpen.apply(this, arguments);
    };
}

function getVisibleCategory() {
    return Array.from(document.getElementById('category-wrap').children).find(child => {
        return child.classList.contains('items-cont') && child.style.display !== 'none';
    });
}

function formatCurrency(value) {
    return '$' + value.toLocaleString('en-US');
}
