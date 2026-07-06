// ==UserScript==
// @name         字节校招全自动海投（含自动翻页）
// @namespace    http://tampermonkey.net/
// @version      11.0
// @description  列表→详情→投递→信息填写→自动翻页循环，全自动
// @match        https://jobs.bytedance.com/campus/position*
// @run-at       document-idle
// @grant        none
// ==/UserScript==

(function() {
    'use strict';

    const CONFIG = {
        jobLinkSelector: 'a[href*="/campus/position/"][href*="/detail"]',
        applyBtnSelector: 'button.apply-block-applyBtn',
        nextPageSelector: 'li.atsx-pagination-next a',   // 翻页按钮选择器
        dailyLimit: 50,
        minDelay: 3000,
        maxDelay: 5000,
    };

    const STORAGE_KEY = 'bytedance_apply_v11';
    const COUNT_KEY = 'bytedance_count_v11';

    const sleep = ms => new Promise(r => setTimeout(r, ms + Math.random() * ms));
    const randomWait = () => sleep(CONFIG.minDelay + Math.random() * (CONFIG.maxDelay - CONFIG.minDelay));

    const getStored = () => JSON.parse(localStorage.getItem(STORAGE_KEY) || '[]');
    const setStored = list => localStorage.setItem(STORAGE_KEY, JSON.stringify(list));
    const getCount = () => parseInt(localStorage.getItem(COUNT_KEY) || '0');
    const addCount = () => localStorage.setItem(COUNT_KEY, getCount() + 1);
    const resetAll = () => {
        localStorage.removeItem(STORAGE_KEY);
        localStorage.removeItem(COUNT_KEY);
    };

    const getJobId = url => (url.match(/\/position\/(\d+)/) || [])[1] || '';

    // 状态条
    function showStatus(text, color = '#4CAF50') {
        let bar = document.getElementById('apply-status-bar');
        if (!bar) {
            bar = document.createElement('div');
            bar.id = 'apply-status-bar';
            bar.style.cssText = 'position:fixed;top:0;left:0;right:0;z-index:99999;padding:10px;color:white;font-size:14px;text-align:center;transition:0.3s;font-family:sans-serif;';
            document.body.appendChild(bar);
        }
        bar.textContent = '🤖 ' + text;
        bar.style.background = color;
    }

    const waitFor = (fn, timeout = 15000) => new Promise(resolve => {
        if (fn()) return resolve(fn());
        const observer = new MutationObserver(() => {
            const el = fn();
            if (el) { observer.disconnect(); resolve(el); }
        });
        observer.observe(document.body, { childList: true, subtree: true });
        const timer = setInterval(() => {
            const el = fn();
            if (el) { observer.disconnect(); clearInterval(timer); resolve(el); }
        }, 500);
        setTimeout(() => {
            observer.disconnect();
            clearInterval(timer);
            resolve(fn() || null);
        }, timeout);
    });

    const findApplyBtn = () => {
        let btn = document.querySelector(CONFIG.applyBtnSelector);
        if (btn) return btn;
        btn = Array.from(document.querySelectorAll('button, a, span, div, [role="button"]'))
                  .find(el => el.innerText.trim() === '投递简历');
        if (btn) return btn;
        btn = Array.from(document.querySelectorAll('button'))
                  .find(el => el.innerText.includes('投递'));
        return btn || null;
    };

    const href = window.location.href;
    const pageText = document.body.innerText;
    const isList = href.includes('/campus/position') && !href.includes('/detail') && !pageText.includes('确认投递') && !pageText.includes('提交简历');
    const isDetail = href.includes('/detail') && !pageText.includes('确认投递') && !pageText.includes('提交简历');
    const isInfoPage = pageText.includes('确认投递') || pageText.includes('提交简历');

    // ==================== 列表页（含翻页） ====================
    if (isList) {
        (async () => {
            // 重置按钮
            const btn = document.createElement('button');
            btn.innerText = '🔄 重置进度';
            btn.style.cssText = 'position:fixed;top:10px;left:10px;z-index:99999;background:#f44336;color:white;border:none;padding:8px 12px;border-radius:5px;font-size:14px;';
            btn.onclick = () => { resetAll(); alert('已重置！刷新页面重新开始'); location.reload(); };
            document.body.appendChild(btn);

            if (getCount() >= CONFIG.dailyLimit) {
                alert('今日已达上限，如需继续请点击重置进度。');
                return;
            }

            // 1. 收集当前页职位链接并与存储合并
            const currentPageLinks = await waitFor(() => {
                const all = document.querySelectorAll(CONFIG.jobLinkSelector);
                return all.length ? Array.from(all) : null;
            }, 20000);

            if (!currentPageLinks || !currentPageLinks.length) {
                // 如果当前页无职位，检查是否有下一页
                const hasNext = await checkAndGoNextPage();
                if (!hasNext) alert('未找到任何职位链接。');
                return;
            }

            const stored = getStored();
            const storedUrls = stored.map(j => j.url);
            currentPageLinks.forEach(a => {
                const url = a.href;
                if (!storedUrls.includes(url)) {
                    stored.push({ url, applied: false });
                }
            });
            setStored(stored);

            console.log(`📦 已存储 ${stored.length} 个职位，当前页未投递：${stored.filter(j => !j.applied).length}`);

            // 2. 检查下一页按钮是否存在且未禁用
            async function checkAndGoNextPage() {
                const nextBtn = document.querySelector(CONFIG.nextPageSelector);
                if (!nextBtn) return false;
                const parentLi = nextBtn.closest('li');
                if (parentLi && parentLi.classList.contains('atsx-pagination-disabled')) return false;
                // 点击翻页
                console.log('➡️ 正在翻到下一页...');
                nextBtn.click();
                // 等待页面跳转（传统分页会刷新，AJAX分页会更新列表）
                await sleep(3000);
                // 如果页面没跳转（AJAX），需要手动等待新链接加载
                return true;
            }

            // 3. 处理单个职位
            async function processNext() {
                const list = getStored();
                const next = list.find(j => !j.applied);
                if (!next) {
                    // 当前列表所有职位已处理完毕，尝试翻页
                    const wentNext = await checkAndGoNextPage();
                    if (wentNext) {
                        // 如果翻页成功（页面刷新或AJAX加载），脚本会重新执行列表页逻辑
                        // 为防止当前脚本继续运行，刷新页面确保重新初始化
                        location.reload();
                    } else {
                        alert('🎉 所有页面职位均已投递完毕！');
                    }
                    return;
                }

                if (getCount() >= CONFIG.dailyLimit) {
                    alert('已达上限');
                    return;
                }

                // 预标记
                next.applied = true;
                setStored(list);

                console.log(`➡️ 打开职位: ${next.url}`);
                const win = window.open(next.url, '_blank');
                if (!win) {
                    next.applied = false;
                    setStored(list);
                    alert('浏览器阻止了弹窗，请允许后刷新页面');
                    return;
                }

                const check = setInterval(() => {
                    if (win.closed) {
                        clearInterval(check);
                        console.log('子窗口已关闭，准备下一个');
                        setTimeout(processNext, 2000);
                    }
                }, 1000);
            }

            processNext();
        })();
    }

    // ==================== 详情页 + 信息填写页监听 ====================
    else if (isDetail || isInfoPage) {
        (async () => {
            if (isInfoPage) {
                await doInfoPageActions();
                return;
            }

            const applyBtn = await waitFor(() => findApplyBtn(), 25000);
            if (!applyBtn) return window.close();
            if (applyBtn.disabled || applyBtn.innerText.includes('已投递')) {
                const list = getStored();
                const id = getJobId(href);
                const job = id ? list.find(j => j.url.includes(id)) : list.find(j => href.includes(j.url));
                if (job) { job.applied = true; setStored(list); }
                addCount();
                return window.close();
            }

            applyBtn.scrollIntoView({ behavior: 'smooth', block: 'center' });
            await randomWait();
            applyBtn.click();
            showStatus('已点击投递，等待信息填写页...', '#2196F3');

            const infoElement = await waitFor(() => {
                const arrow = document.querySelector('.ud__select__selector__arrow');
                const confirmBtn = Array.from(document.querySelectorAll('button'))
                    .find(b => b.innerText.trim() === '确认投递' || b.innerText.trim() === '提交简历');
                return arrow || confirmBtn;
            }, 30000);

            if (!infoElement) {
                showStatus('❌ 信息填写页未出现', '#f44336');
                await sleep(5000);
                return window.close();
            }

            await doInfoPageActions();
        })();
    }

    async function doInfoPageActions() {
        showStatus('开始填写信息...', '#FF9800');

        const targetLabelText = '你从哪里获知字节跳动校招信息';
        const labelEl = await waitFor(() => {
            const labels = document.querySelectorAll('.ud-formily-item-label');
            return Array.from(labels).find(l => l.innerText.includes(targetLabelText));
        }, 10000);

        if (!labelEl) {
            showStatus('❌ 未找到文本：' + targetLabelText, '#f44336');
            return;
        }

        const formItem = labelEl.closest('.ud-formily-item');
        if (!formItem) {
            showStatus('❌ 无法定位表单容器', '#f44336');
            return;
        }

        let clickTarget = formItem.querySelector('.ud__select__selector') || formItem.querySelector('.ud__select');
        if (!clickTarget) {
            showStatus('❌ 未找到下拉控件', '#f44336');
            return;
        }

        clickTarget.scrollIntoView({ behavior: 'smooth', block: 'center' });
        await sleep(300);
        clickTarget.click();
        await sleep(500);

        let menuOpen = document.querySelector('.ud__select__dropdown') !== null;
        if (!menuOpen) {
            clickTarget.dispatchEvent(new MouseEvent('mousedown', { bubbles: true }));
            await sleep(500);
            menuOpen = document.querySelector('.ud__select__dropdown') !== null;
        }
        if (!menuOpen) {
            showStatus('❌ 菜单未能打开，请手动点击下拉框', '#f44336');
            return;
        }

        showStatus('已打开下拉菜单，选择“其他”...', '#FF9800');

        const other = await waitFor(() => {
            const contents = document.querySelectorAll('div.ud__select__list__item__content');
            return Array.from(contents).find(el => el.innerText.trim() === '其他');
        }, 8000);
        if (!other) {
            showStatus('❌ 未找到“其他”选项', '#f44336');
            return;
        }
        (other.closest('.ud__select__list__item') || other).click();
        showStatus('已选择“其他”，点击提交...', '#FF9800');

        const submitBtn = await waitFor(() => {
            const btns = document.querySelectorAll('button.atsx-btn-primary.atsx-btn-lg');
            return Array.from(btns).find(b => b.innerText.trim() === '确认投递' || b.innerText.trim() === '提交简历');
        }, 10000);
        if (!submitBtn) {
            showStatus('❌ 未找到提交按钮', '#f44336');
            return;
        }
        submitBtn.scrollIntoView({ behavior: 'smooth', block: 'center' });
        await randomWait();
        submitBtn.click();
        showStatus('已提交简历，即将关闭...', '#4CAF50');

        await sleep(3000);

        const currentUrl = window.location.href;
        const list = getStored();
        const id = getJobId(currentUrl);
        const job = id ? list.find(j => j.url.includes(id)) : list.find(j => currentUrl.includes(j.url));
        if (job) { job.applied = true; setStored(list); }
        addCount();

        await sleep(1000);
        window.close();
    }
})();
