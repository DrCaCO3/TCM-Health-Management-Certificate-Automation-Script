# TCM-Health-Management-Certificate-Automation-Script
留给后人的小小帮助
7倍速播放+自动连播
写在前面
建议在电脑上尝试
因为不确定手机游览器是否能安装相同插件
所以只提供Edge游览器的教程
本脚本完全由https://github.com/deepseek-ai/deepseek-harness 完成
模型为V4Pro花费约1RNB
小概率抽风表现为无法正常点击下一章这时需要手动点击（会修的？）
有技术性问题交Insigts（有空看）
正常问题请自行搜索
# 正文
首先在Edgey游览器上安装 Global Speed: 视频速度控制 和 篡改猴（Tampermonkey）扩展
Ps: 如果你只需要倍速播放只需要安装 Global Speed: 视频速度控制 就可以了（倍速过高会有反效果）
然后在管理扩展里面点击 篡改猴 的详细信息
找到 允许用户脚本 打开它
然后打开证书网站的视频界面调整倍速（建议不要大于7.5）
打开 篡改猴 的界面，添加新脚本
把原本的模板删除，复制粘贴下面的脚本 ctrl+S 保存，启用脚本后ctrl+F5刷新就完成了
# 脚本
// ==UserScript==
// @name         视频自动下一章 v1.3
// @namespace    xjn-auto-next
// @version      1.3
// @description  播完→等进度保存→立即刷新→快速点击“下一章”（优化执行时长）
// @author       you
// @match        *://*/*
// @grant        none
// @run-at       document-end
// @all-frames   true
// ==/UserScript==

(function () {
  'use strict';
  if (!/ethrss/i.test(location.hostname)) return;

  // ====== 可调参数 ======
  const FLAG = 'xjn_auto_reload_done';
  const WAIT_AFTER_END = 4000;         // 播完后等网站保存“已看完”进度的时间（毫秒）
  const TRY_BEFORE_REFRESH = false;    // 刷新前是否快速试一次点击（已知无效，默认关；开启只多花约1.5秒）
  const CLICK_VERIFY_MS = 1200;        // 点击后验证跳转的等待时间（已从2500缩短）
  const SCROLL_WAIT_MS = 200;          // 滚动后等待时间
  const REFRESH_DELAY_MS = 300;        // 刷新前最后等待
  const POLL_MS = 500;                 // 找按钮的轮询间隔
  const MAX_AFTER_REFRESH_WAIT = 30000;// 刷新后最多等按钮多久
  const PAUSE_AFTER_REFRESH = true;    // 刷新后先暂停视频，防止自动重播
  const DISMISS_DIALOG = true;         // 自动关掉“知道了/确定”之类的弹窗

  const isTop = window === window.top;
  const log = (...a) => console.log('[自动下一章]', ...a);
  const sleep = (ms) => new Promise(r => setTimeout(r, ms));
  let busy = false;

  // ---------- 左上角状态角标 ----------
  function badge(text, color) {
    let el = document.getElementById('xjn-auto-badge');
    if (!el) {
      el = document.createElement('div');
      el.id = 'xjn-auto-badge';
      el.style.cssText = 'position:fixed;top:10px;left:10px;z-index:2147483647;'
        + 'background:' + (color || '#e65100') + ';color:#fff;padding:4px 10px;'
        + 'border-radius:4px;font-size:12px;font-family:sans-serif;pointer-events:none;';
      document.body.appendChild(el);
    }
    if (color) el.style.background = color;
    el.textContent = text;
  }

  function getVideo() { return document.querySelector('video'); }

  // ---------- 页面指纹：URL + 视频地址 + 标题，判断点击后是否真的跳转 ----------
  function pageSnapshot() {
    const v = getVideo();
    return location.href + '|' + (v ? (v.currentSrc || v.src || '') : '') + '|' + document.title;
  }

  // ---------- 自动关弹窗 ----------
  function dismissDialogs() {
    if (!DISMISS_DIALOG) return;
    const texts = ['知道了', '我知道了', '确定', '好的', '关闭', '继续学习', '进入下一章'];
    for (const t of texts) {
      const btn = [...document.querySelectorAll('button, a, div, span')]
        .find(el => (el.textContent || '').trim() === t && el.offsetParent !== null);
      if (btn) { try { btn.click(); } catch (e) {} return; }
    }
  }

  // ---------- 找“下一章”：越靠近文字的内层元素越优先 ----------
  function findNextCandidates() {
    const list = [];
    const seen = new Set();
    for (const el of document.querySelectorAll('a, button, [role="button"], div, span')) {
      const t = el.textContent || '';
      if (!/下一章/.test(t)) continue;
      if (el.offsetParent === null) continue;          // 跳过隐藏元素
      if (seen.has(el)) continue;
      seen.add(el);
      list.push({ el, depth: el.querySelectorAll('*').length, text: t.trim().slice(0, 50) });
    }
    list.sort((a, b) => a.depth - b.depth);            // 深层（具体文字）优先
    return list;
  }

  // ---------- 点击候选元素并验证（每次点击后只等1.2秒） ----------
  async function tryClickNext(cands) {
    dismissDialogs();
    const before = pageSnapshot();

    for (const { el, text } of cands.slice(0, 6)) {
      log('点击:', text, '<' + el.tagName + '>');
      try { el.scrollIntoView({ behavior: 'smooth', block: 'center' }); } catch (e) {}
      await sleep(SCROLL_WAIT_MS);
      try {
        el.dispatchEvent(new MouseEvent('mousedown', { bubbles: true, cancelable: true, view: window }));
        el.dispatchEvent(new MouseEvent('mouseup', { bubbles: true, cancelable: true, view: window }));
        el.click();
      } catch (e) { log('点击异常', e); }
      await sleep(CLICK_VERIFY_MS);
      if (pageSnapshot() !== before) { log('✅ 页面已变化'); return { ok: true }; }
    }

    // 兜底：真实链接直接同标签页跳转
    const links = [...document.querySelectorAll('a')]
      .filter(a => /下一章/.test(a.textContent || '') && a.href && !a.href.startsWith('javascript:'));
    for (const a of links.slice(0, 2)) {
      log('直接跳转链接:', a.href);
      try { location.assign(a.href); } catch (e) {}
      await sleep(CLICK_VERIFY_MS);
      if (pageSnapshot() !== before) { log('✅ 链接跳转成功'); return { ok: true }; }
    }

    return { ok: false };
  }

  // ---------- 视频结束后的主流程 ----------
  async function handleVideoEnded() {
    if (busy) return;
    busy = true;
    const t0 = Date.now();
    log('视频播放结束');
    badge('播放完毕，等待进度保存…', '#e65100');

    await sleep(WAIT_AFTER_END);          // 给网站时间保存完成状态
    dismissDialogs();

    // 可选：刷新前快速试一次（默认关闭，节省时间）
    if (TRY_BEFORE_REFRESH) {
      const cands = findNextCandidates();
      if (cands.length) {
        badge('快速尝试点击…', '#1565c0');
        const r = await tryClickNext(cands.slice(0, 2));
        if (r.ok) { badge('已进入下一章 ✓', '#2e7d32'); log('总耗时', Date.now() - t0, 'ms'); busy = false; return; }
      }
    }

    // 立即刷新
    log('刷新页面（已耗时', Date.now() - t0, 'ms）');
    badge('刷新页面…', '#c62828');
    sessionStorage.setItem(FLAG, '1');
    await sleep(REFRESH_DELAY_MS);
    location.reload();
  }

  // ---------- 刷新后：暂停视频，快速点“下一章” ----------
  async function handleAfterRefresh() {
    sessionStorage.removeItem(FLAG);
    const t0 = Date.now();
    log('刷新后进入，寻找“下一章”');
    badge('已刷新，寻找“下一章”…', '#1565c0');

    const v = getVideo();
    if (v && PAUSE_AFTER_REFRESH) { try { v.pause(); } catch (e) {} }   // 防止自动重播

    const deadline = Date.now() + MAX_AFTER_REFRESH_WAIT;
    while (Date.now() < deadline) {
      const cands = findNextCandidates();
      if (cands.length) {
        const r = await tryClickNext(cands);
        if (r.ok) {
          badge('已进入下一章 ✓', '#2e7d32');
          log('刷新后耗时', Date.now() - t0, 'ms');
          return;
        }
        await sleep(1000);
      } else {
        await sleep(POLL_MS);
      }
    }

    // 失败：恢复播放，视频再播完一次后自动重试
    log('点击未成功，恢复播放等待下次结束重试');
    badge('未成功，等待视频再次播完重试', '#c62828');
    if (v) { try { v.play(); } catch (e) {} }
    watchVideo(handleVideoEnded, true);
  }

  // ---------- 监听当前帧的视频 ----------
  function watchVideo(onEnded, isTopFrame) {
    const v = document.querySelector('video');
    if (!v) {
      const ob = new MutationObserver(() => {
        if (document.querySelector('video')) { ob.disconnect(); watchVideo(onEnded, isTopFrame); }
      });
      ob.observe(document.body, { childList: true, subtree: true });
      return;
    }
    if (v.loop) { v.loop = false; log('已关闭视频循环播放'); }
    if (isTopFrame) {
      log('找到视频元素，时长', (v.duration || '未知'), '秒');
      badge('视频播放中…', '#2e7d32');
    }
    v.addEventListener('ended', onEnded);
    const timer = setInterval(() => {
      if (v.duration && v.currentTime >= v.duration - 0.15) {
        clearInterval(timer);
        onEnded();
      }
    }, 1000);
  }

  // ---------- 启动 ----------
  if (isTop) {
    log('脚本已启动（顶层页面）');
    badge('自动下一章已启动', '#e65100');

    window.addEventListener('message', (e) => {
      if (e.data && e.data.type === 'XJN_VIDEO_ENDED') handleVideoEnded();
    });

    if (sessionStorage.getItem(FLAG) === '1') {
      handleAfterRefresh();                 // 刷新后 → 点下一章
    } else {
      watchVideo(handleVideoEnded, true);   // 正常 → 等视频播完
    }
  } else {
    // iframe 里的视频：结束就通知顶层页面
    const send = () => {
      try { window.top.postMessage({ type: 'XJN_VIDEO_ENDED' }, '*'); } catch (e) {}
    };
    watchVideo(send, false);
  }
})();
# 结语
希望能帮上有需求的人
26.08.25.07.05
