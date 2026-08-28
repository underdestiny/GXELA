// ==UserScript==
// @name         GXELA 自动选课 + 自动学习（V3.6）
// @namespace    http://tampermonkey.net/
// @version      3.6
// @description  GXELA 自动学习、班级必修同步、完成确认、队列恢复与日志筛选
// @author       GXELA
// @match        https://www.gxela.gov.cn/*
// @grant        GM_setValue
// @grant        GM_getValue
// @grant        GM_deleteValue
// @grant        GM_xmlhttpRequest
// @grant        GM_addStyle
// @downloadURL  https://cdn.jsdelivr.net/gh/underdestiny/GXELA@main/GXELA%20%E8%87%AA%E5%8A%A8%E9%80%89%E8%AF%BE%20+%20%E8%87%AA%E5%8A%A8%E5%AD%A6%E4%B9%A0%EF%BC%88v3.6%EF%BC%89.user.js
// @updateURL    https://cdn.jsdelivr.net/gh/underdestiny/GXELA@main/GXELA%20%E8%87%AA%E5%8A%A8%E9%80%89%E8%AF%BE%20+%20%E8%87%AA%E5%8A%A8%E5%AD%A6%E4%B9%A0%EF%BC%88v3.6%EF%BC%89.user.js

// ==/UserScript==

(function () {
    'use strict';

    /********************************************************************
     * GXELA 自动选课 + 自动学习
     * Version: V3.6
     ********************************************************************/

    /* ================================================================
     * 配置
     * ================================================================ */

    const FALLBACK_CLIENT_ID = 'e5cd7e4891bf95d1d19206ce24a7b32e';

    const PAGE_SIZE = 100;

    const SELECT_DELAY_MS = 600;
    const CANCEL_DELAY_MS = 400;

    const SELECT_MAX_RETRIES = 2;
    const CANCEL_MAX_RETRIES = 2;

    const LOG_MAX_LINES = 1500;

    // video 查找
    const VIDEO_DETECT_RETRY_MS = 5000;
    const VIDEO_DETECT_MAX_RETRIES = 12;

    // 卡顿检测
    const VIDEO_STUCK_CHECK_MS = 15000;
    const VIDEO_STUCK_THRESHOLD = 3;

    // 页面重载次数
    const VIDEO_MAX_RELOADS = 2;

    // 自动播放重试
    const PLAY_RETRY_INTERVAL = 2500;
    const PLAY_MAX_RETRIES = 8;

    // 失焦恢复
    const BLUR_RECOVER_DELAY = 800;

    // 页面点击消除弹窗
    const DIALOG_DISMISS_DELAY = 800;


    /* ================================================================
     * 持久化
     * ================================================================ */

    function saveState(key, value) {
        try {
            GM_setValue(key, value);
        } catch (e) {
            console.warn('[GXELA] saveState error:', e);
        }
    }

    function loadState(key, def) {
        try {
            const value = GM_getValue(key);
            return value === undefined ? def : value;
        } catch (e) {
            return def;
        }
    }


    /* ================================================================
     * 运行状态
     * ================================================================ */

    let logs = loadState('gxela_logs', []);

    let logFilter = loadState('gxela_logFilter', 'all');

    let targetHours = loadState(
        'gxela_targets',
        {
            compulsory: 0,
            elective: 0
        }
    );

    let pendingIdsSet = new Set(
        loadState('gxela_pendingIds', [])
    );

    let clientIdCurrent =
        loadState('gxela_clientid', null);

    let clientIdSource =
        loadState('gxela_clientid_source', null);

    let currentYear =
        loadState('gxela_year', new Date().getFullYear());

    let videoPlayRetryCount = 0;

    let videoMonitorStarted = false;

    let currentVideo = null;

    let pageClickTimer = null;

    let classSyncState = loadState('gxela_classSyncState', {
        active: false,
        phase: null,
        pending: [],
        failed: [],
        current: null,
        results: []
    });


    /* ================================================================
     * 日志
     * ================================================================ */

    function nowTime() {
        return new Date().toLocaleTimeString();
    }

    function appendLog(text) {

        const line =
            `[${nowTime()}] ${text}`;

        logs.push(line);

        if (logs.length > LOG_MAX_LINES) {
            logs = logs.slice(-LOG_MAX_LINES);
        }

        saveState('gxela_logs', logs);

        renderLogs(true);

        console.log(line);
    }


    function renderLogs(scrollToEnd = false) {

        const box = document.getElementById('gxela-logbox');

        if (!box) return;

        const visible = logs.filter(line => {
            if (logFilter === 'warning') return /⚠|❌|失败|异常|卡顿|跳过/.test(line);
            if (logFilter === 'success') return /✅|🎉|成功|完成|已恢复/.test(line);
            return true;
        });

        box.textContent = visible.join('\n');

        if (scrollToEnd) box.scrollTop = box.scrollHeight;

    }


    function sleep(ms) {
        return new Promise(resolve => setTimeout(resolve, ms));
    }


    /* ================================================================
     * Cookie Authorization
     * ================================================================ */

    function getAuthFromCookie() {

        const match =
            document.cookie.match(/GXELA-Token=([^;]+)/);

        return match
            ? `Bearer ${match[1]}`
            : '';
    }


    /* ================================================================
     * clientid
     * ================================================================ */

    function saveClientId(cid, source) {

        if (!cid) return;

        cid = String(cid).trim();

        clientIdCurrent = cid;

        clientIdSource =
            source || 'auto';

        saveState(
            'gxela_clientid',
            cid
        );

        saveState(
            'gxela_clientid_source',
            clientIdSource
        );

        appendLog(
            `clientid 已保存（来源:${clientIdSource}）`
        );

        const input =
            document.getElementById(
                'gxela-clientid-input'
            );

        const status =
            document.getElementById(
                'gxela-clientid-status'
            );

        if (input) {
            input.value = cid;
        }

        if (status) {
            status.textContent =
                `已设置 · ${cid}`;
        }

        updateStartButton();
    }


    function clearClientId() {

        clientIdCurrent = null;
        clientIdSource = null;

        saveState(
            'gxela_clientid',
            null
        );

        saveState(
            'gxela_clientid_source',
            null
        );

        const status =
            document.getElementById(
                'gxela-clientid-status'
            );

        if (status) {
            status.textContent =
                '尚未设置 clientid';
        }

        updateStartButton();

        appendLog(
            'clientid 已清除'
        );
    }


    function updateStartButton() {

        const button =
            document.getElementById(
                'gxela-startBtn'
            );

        if (button) {
            button.disabled =
                !clientIdCurrent;
        }
    }


    function getClientIdForHeader() {
        return clientIdCurrent || null;
    }


    /* ================================================================
     * GM 网络请求
     * ================================================================ */

    function gmRequestRaw(options) {

        return new Promise((resolve, reject) => {

            try {

                options =
                    Object.assign({}, options);

                options.headers =
                    Object.assign(
                        {},
                        options.headers || {}
                    );

                const cid =
                    getClientIdForHeader();

                if (
                    cid &&
                    !options.headers.clientid
                ) {
                    options.headers.clientid =
                        cid;
                }

                const auth =
                    getAuthFromCookie();

                if (
                    auth &&
                    !options.headers.Authorization
                ) {
                    options.headers.Authorization =
                        auth;
                }

                GM_xmlhttpRequest(
                    Object.assign(
                        {},
                        options,
                        {
                            onload(res) {
                                resolve(res);
                            },

                            onerror(err) {
                                reject(err);
                            }
                        }
                    )
                );

            } catch (e) {

                reject(e);

            }

        });
    }


    /* ================================================================
     * clientid 页面拦截
     * ================================================================ */

    function injectClientIdInterceptorToPage() {

        try {

            const scriptContent = `
                (function () {

                    try {

                        const originalFetch = window.fetch;

                        window.fetch = function (input, init) {

                            try {

                                let headers =
                                    init && init.headers
                                        ? init.headers
                                        : null;

                                let cid = null;

                                if (headers) {

                                    if (
                                        typeof headers.get === 'function'
                                    ) {

                                        cid =
                                            headers.get('clientid') ||
                                            headers.get('ClientId');

                                    }

                                    else if (
                                        Array.isArray(headers)
                                    ) {

                                        for (
                                            const h of headers
                                        ) {

                                            if (
                                                h[0] &&
                                                h[0].toLowerCase() === 'clientid'
                                            ) {

                                                cid = h[1];
                                                break;

                                            }

                                        }

                                    }

                                    else if (
                                        typeof headers === 'object'
                                    ) {

                                        for (
                                            const key in headers
                                        ) {

                                            if (
                                                key &&
                                                key.toLowerCase() === 'clientid'
                                            ) {

                                                cid =
                                                    headers[key];

                                                break;

                                            }

                                        }

                                    }

                                }

                                if (cid) {

                                    window.postMessage(
                                        {
                                            type:
                                                'GXELA_CLIENTID',
                                            clientid:
                                                cid
                                        },
                                        '*'
                                    );

                                }

                            } catch (e) {}

                            return originalFetch.apply(
                                this,
                                arguments
                            );

                        };


                        const originalSetHeader =
                            XMLHttpRequest
                                .prototype
                                .setRequestHeader;


                        XMLHttpRequest
                            .prototype
                            .setRequestHeader =
                            function (
                                header,
                                value
                            ) {

                                try {

                                    if (
                                        header &&
                                        header.toLowerCase() ===
                                            'clientid'
                                    ) {

                                        window.postMessage(
                                            {
                                                type:
                                                    'GXELA_CLIENTID',
                                                clientid:
                                                    value
                                            },
                                            '*'
                                        );

                                    }

                                } catch (e) {}

                                return originalSetHeader.apply(
                                    this,
                                    arguments
                                );

                            };


                        try {

                            if (window.clientid) {

                                window.postMessage(
                                    {
                                        type:
                                            'GXELA_CLIENTID',
                                        clientid:
                                            window.clientid
                                    },
                                    '*'
                                );

                            }

                            if (window.__CLIENTID__) {

                                window.postMessage(
                                    {
                                        type:
                                            'GXELA_CLIENTID',
                                        clientid:
                                            window.__CLIENTID__
                                    },
                                    '*'
                                );

                            }

                        } catch (e) {}

                    } catch (e) {}

                })();
            `;

            const script =
                document.createElement('script');

            script.type =
                'text/javascript';

            script.textContent =
                scriptContent;

            (
                document.head ||
                document.documentElement
            ).appendChild(script);

            script.remove();

            appendLog(
                '已注入 clientid 页面拦截器'
            );

        } catch (e) {

            appendLog(
                'clientid 拦截器注入失败：' +
                e
            );

        }
    }


    window.addEventListener(
        'message',
        function (event) {

            try {

                const data =
                    event.data;

                if (
                    data &&
                    data.type ===
                        'GXELA_CLIENTID' &&
                    data.clientid
                ) {

                    const cid =
                        String(
                            data.clientid
                        ).trim();

                    if (cid.length >= 8) {

                        saveClientId(
                            cid,
                            'auto'
                        );

                    }

                }

            } catch (e) {}

        }
    );


    function scanPageForClientId() {

        try {

            const metas =
                document.querySelectorAll(
                    'meta'
                );

            for (
                const meta of metas
            ) {

                const name =
                    (
                        meta.getAttribute('name') ||
                        ''
                    ).toLowerCase();

                const content =
                    (
                        meta.getAttribute('content') ||
                        ''
                    ).trim();

                if (
                    name.includes('clientid') &&
                    content
                ) {
                    return content;
                }

                const match =
                    content.match(
                        /clientid\s*[:=]\s*['"]?([a-z0-9_-]{8,64})['"]?/i
                    );

                if (match) {
                    return match[1];
                }

            }


            const scripts =
                document.getElementsByTagName(
                    'script'
                );

            const regex =
                /clientid\s*[:=]\s*['"]?([a-z0-9_-]{8,64})['"]?/i;


            for (
                const script of scripts
            ) {

                const text =
                    script.textContent || '';

                const match =
                    text.match(regex);

                if (
                    match &&
                    match[1]
                ) {

                    return match[1];

                }

            }

        } catch (e) {

            console.warn(e);

        }

        return null;
    }


    /* ================================================================
     * UI 样式
     * ================================================================ */

    GM_addStyle(`

        #gxela-panel {

            position: fixed;

            top: 18px;
            right: 18px;

            width: 540px;

            max-height: calc(100vh - 36px);

            z-index: 2147483647;

            display: flex;
            flex-direction: column;

            background: #f5f7fa;

            border: 1px solid #dfe3e8;

            border-radius: 16px;

            box-shadow:
                0 12px 40px rgba(0,0,0,.16);

            overflow: hidden;

            font-family:
                -apple-system,
                BlinkMacSystemFont,
                "Segoe UI",
                "Microsoft YaHei",
                Arial,
                sans-serif;

            color: #202124;

            font-size: 13px;

        }


        #gxela-header {

            flex-shrink: 0;

            padding: 16px 18px;

            color: white;

            background:
                linear-gradient(
                    135deg,
                    #2563eb,
                    #4f46e5
                );

            display: flex;

            justify-content:
                space-between;

            align-items: center;

            cursor: pointer;

            user-select: none;

        }


        #gxela-header-title {

            display: flex;

            align-items: center;

            gap: 10px;

            font-size: 16px;

            font-weight: 700;

        }


        #gxela-header-sub {

            margin-top: 3px;

            opacity: .78;

            font-size: 11px;

        }


        #gxela-header-arrow {

            font-size: 18px;

            opacity: .85;

        }


        #gxela-body {

            overflow-y: auto;

            padding: 12px;

        }


        #gxela-body.collapsed {

            display: none;

        }


        .gxela-card {

            margin-bottom: 10px;

            background: white;

            border: 1px solid #e5e7eb;

            border-radius: 12px;

            overflow: hidden;

        }


        .gxela-card:last-child {

            margin-bottom: 0;

        }


        .gxela-card-title {

            min-height: 42px;

            padding: 0 13px;

            display: flex;

            align-items: center;

            justify-content: space-between;

            background: #fff;

            cursor: pointer;

            user-select: none;

            font-weight: 700;

        }


        .gxela-card-title:hover {

            background: #f8fafc;

        }


        .gxela-card-title-left {

            display: flex;

            align-items: center;

            gap: 8px;

        }


        .gxela-card-icon {

            width: 25px;
            height: 25px;

            display: flex;
            align-items: center;
            justify-content: center;

            border-radius: 7px;

            background: #eef2ff;

        }


        .gxela-card-arrow {

            color: #9ca3af;

        }


        .gxela-card-content {

            padding: 12px;

            border-top: 1px solid #f0f1f3;

        }


        .gxela-card-content.collapsed {

            display: none;

        }


        /* 目标学时 */

        .gxela-target-grid {

            display: grid;

            grid-template-columns:
                1fr 1fr;

            gap: 10px;

        }


        .gxela-target-box {

            padding: 14px;

            border-radius: 11px;

            background:
                linear-gradient(
                    135deg,
                    #eff6ff,
                    #eef2ff
                );

            border: 1px solid #dbeafe;

        }


        .gxela-target-label {

            font-size: 12px;

            color: #64748b;

        }


        .gxela-target-input {

            width: 100%;

            box-sizing: border-box;

            margin-top: 6px;

            padding: 7px 8px;

            border: 1px solid #cbd5e1;

            border-radius: 7px;

            font-size: 18px;

            font-weight: 700;

            background: white;

        }


        .gxela-stat-grid {

            display: grid;

            grid-template-columns:
                repeat(4, 1fr);

            gap: 7px;

            margin-top: 10px;

        }


        .gxela-stat {

            padding: 9px 5px;

            text-align: center;

            border-radius: 9px;

            background: #f8fafc;

            border: 1px solid #eef0f2;

        }


        .gxela-stat-value {

            font-size: 16px;

            font-weight: 700;

        }


        .gxela-stat-label {

            margin-top: 2px;

            font-size: 10px;

            color: #64748b;

        }


        .gxela-remaining {

            margin-top: 10px;

            padding: 10px;

            border-radius: 9px;

            background: #fefce8;

            border: 1px solid #fef08a;

        }


        .gxela-remaining-row {

            display: flex;

            justify-content:
                space-between;

            padding: 2px 0;

        }


        /* 按钮 */

        .gxela-buttons {

            display: flex;

            flex-wrap: wrap;

            gap: 7px;

        }


        .gxela-btn {

            border: 0;

            border-radius: 7px;

            padding: 7px 11px;

            cursor: pointer;

            background: #f1f5f9;

            color: #334155;

            font-size: 12px;

            transition: .15s;

        }


        .gxela-btn:hover {

            background: #e2e8f0;

        }


        .gxela-btn.primary {

            color: white;

            background:
                linear-gradient(
                    135deg,
                    #2563eb,
                    #4f46e5
                );

        }


        .gxela-btn.danger {

            color: #b91c1c;

            background: #fef2f2;

        }


        .gxela-btn:disabled {

            opacity: .45;

            cursor: not-allowed;

        }


        /* 当前播放 */

        .gxela-playing {

            padding: 12px;

            border-radius: 10px;

            background:
                linear-gradient(
                    135deg,
                    #eff6ff,
                    #eef2ff
                );

            border: 1px solid #dbeafe;

        }


        .gxela-playing-name {

            font-size: 14px;

            font-weight: 700;

            line-height: 1.5;

        }


        .gxela-playing-meta {

            margin-top: 5px;

            color: #64748b;

            font-size: 11px;

        }


        .gxela-next {

            margin-top: 8px;

            color: #64748b;

            font-size: 11px;

        }


        /* 课程列表 */

        #gxela-course-list {

            max-height: 390px;

            overflow-y: auto;

            padding-right: 2px;

        }


        .gxela-course-item {

            display: grid;

            grid-template-columns:
                34px 1fr auto;

            gap: 8px;

            align-items: center;

            padding: 9px 6px;

            border-bottom:
                1px solid #f1f5f9;

        }


        .gxela-course-item:last-child {

            border-bottom: 0;

        }


        .gxela-course-index {

            width: 28px;
            height: 28px;

            display: flex;

            align-items: center;

            justify-content: center;

            border-radius: 7px;

            background: #f1f5f9;

            color: #64748b;

            font-size: 11px;

            font-weight: 700;

        }


        .gxela-course-main {

            min-width: 0;

        }


        .gxela-course-name {

            overflow: hidden;

            white-space: nowrap;

            text-overflow: ellipsis;

            font-weight: 600;

        }


        .gxela-course-meta {

            margin-top: 3px;

            color: #94a3b8;

            font-size: 10px;

        }


        .gxela-course-play {

            padding: 5px 9px;

            border: 0;

            border-radius: 6px;

            color: white;

            background: #2563eb;

            cursor: pointer;

            font-size: 11px;

        }


        .gxela-course-play:hover {

            background: #1d4ed8;

        }


        .gxela-course-current {

            background: #eff6ff;

        }


        .gxela-course-current
        .gxela-course-index {

            color: white;

            background: #2563eb;

        }


        .gxela-empty {

            padding: 30px 10px;

            text-align: center;

            color: #94a3b8;

        }


        /* clientid */

        .gxela-client-status {

            margin-bottom: 8px;

            padding: 8px;

            border-radius: 7px;

            background: #f8fafc;

            color: #64748b;

            font-size: 11px;

            word-break: break-all;

        }


        .gxela-client-input {

            width: 100%;

            box-sizing: border-box;

            padding: 7px 9px;

            border: 1px solid #cbd5e1;

            border-radius: 7px;

            margin-bottom: 7px;

        }


        /* 日志 */

        #gxela-logbox {

            height: 190px;

            overflow: auto;

            padding: 8px;

            border-radius: 8px;

            background: #111827;

            color: #d1d5db;

            font-family:
                Consolas,
                "Courier New",
                monospace;

            font-size: 10px;

            line-height: 1.55;

            white-space: pre-wrap;

        }


        /* 状态 */

        #gxela-status {

            margin-bottom: 10px;

            padding: 9px 10px;

            border-radius: 8px;

            background: #f8fafc;

            color: #475569;

            font-size: 11px;

            font-weight: 600;

        }


        .gxela-year-row {

            display: flex;

            align-items: center;

            gap: 7px;

            margin-bottom: 10px;

        }


        .gxela-year-input {

            width: 100px;

            padding: 6px 8px;

            border: 1px solid #cbd5e1;

            border-radius: 7px;

        }


        @media (max-width: 700px) {

            #gxela-panel {

                width: calc(100vw - 20px);

                right: 10px;

                top: 10px;

            }

        }

    `);


    /* V3.6：强化信息层级、班级同步、队列操作与移动端排版。 */
    GM_addStyle(`
        #gxela-panel { width: 570px; border: 1px solid #dbe4f0; background: #f8fafc; }
        #gxela-header { background: linear-gradient(135deg, #0f4c81, #2563eb 62%, #7c3aed); }
        #gxela-body { padding: 10px; scrollbar-gutter: stable; }
        .gxela-card { border: 1px solid #e2e8f0; border-radius: 12px; box-shadow: 0 2px 8px rgba(15, 23, 42, .04); overflow: hidden; }
        .gxela-card-title { min-height: 42px; background: #fff; }
        .gxela-card-title:hover { background: #f8fafc; }
        .gxela-card-content { background: #fff; }
        .gxela-target-card { border: 2px solid #93c5fd; box-shadow: 0 8px 22px rgba(37, 99, 235, .13); }
        .gxela-target-card .gxela-card-title { background: linear-gradient(90deg, #eff6ff, #fff); font-size: 15px; }
        .gxela-target-input { font-size: 22px; font-weight: 750; color: #1d4ed8; background: #f8fbff; }
        .gxela-target-box { padding: 12px; border-radius: 10px; background: #f8fafc; }
        .gxela-target-hint { margin: 10px 0 2px; color: #64748b; font-size: 11px; }
        #gxela-course-list { max-height: min(48vh, 520px); overflow-y: auto; overscroll-behavior: contain; padding: 2px 5px 2px 1px; }
        #gxela-course-list::-webkit-scrollbar { width: 9px; }
        #gxela-course-list::-webkit-scrollbar-thumb { background: #cbd5e1; border-radius: 99px; border: 2px solid #fff; }
        .gxela-course-item { margin-bottom: 7px; border-radius: 9px; }
        .gxela-course-play { white-space: nowrap; }
        .gxela-course-actions { display: flex; gap: 4px; align-items: center; }
        .gxela-course-action { border: 1px solid #cbd5e1; background: #fff; color: #475569; border-radius: 6px; padding: 4px 6px; font-size: 11px; cursor: pointer; }
        .gxela-course-action:hover { border-color: #2563eb; color: #1d4ed8; background: #eff6ff; }
        .gxela-queue-controls { margin: 0 0 9px; display: flex; gap: 7px; flex-wrap: wrap; }
        .gxela-log-tools { display: flex; gap: 7px; margin: 0 0 8px; }
        #gxela-logFilter { flex: 1; border: 1px solid #cbd5e1; border-radius: 7px; color: #334155; padding: 5px 7px; }
        .gxela-queue-hint { margin: 0 0 9px; color: #64748b; font-size: 11px; }
        @media (max-width: 700px) { #gxela-panel { width: calc(100vw - 16px); right: 8px; top: 8px; } #gxela-course-list { max-height: 40vh; } }
    `);

    /* ================================================================
     * UI 创建
     * ================================================================ */

    function createPanel() {

        if (
            document.getElementById(
                'gxela-panel'
            )
        ) {
            return;
        }


        const panel =
            document.createElement('div');

        panel.id =
            'gxela-panel';


        panel.innerHTML = `

            <div id="gxela-header">

                <div>

                    <div id="gxela-header-title">
                        📚 GXELA 自动学习
                    </div>

                    <div id="gxela-header-sub">
                        V3.6 · 班级同步 · 完成确认 · 队列管理
                    </div>

                </div>

                <div id="gxela-header-arrow">
                    ▼
                </div>

            </div>


            <div id="gxela-body">


                <!-- 目标学时 -->

                <div class="gxela-card gxela-target-card">

                    <div
                        class="gxela-card-title"
                        data-target="target-card"
                    >

                        <div class="gxela-card-title-left">

                            <span class="gxela-card-icon">
                                🎯
                            </span>

                            <span>
                                目标学时
                            </span>

                        </div>

                        <span class="gxela-card-arrow">
                            ▼
                        </span>

                    </div>


                    <div
                        id="target-card"
                        class="gxela-card-content"
                    >

                        <div class="gxela-year-row">

                            <span>
                                学习年份
                            </span>

                            <input
                                id="gxela-year"
                                class="gxela-year-input"
                                type="number"
                                min="2000"
                                value="${currentYear}"
                            >

                        </div>


                        <div class="gxela-target-grid">

                            <div class="gxela-target-box">

                                <div class="gxela-target-label">
                                    必修目标学时
                                </div>

                                <input
                                    id="gxela-targetCompulsory"
                                    class="gxela-target-input"
                                    type="number"
                                    step="0.25"
                                    min="0"
                                    value="${targetHours.compulsory || 0}"
                                >

                            </div>


                            <div class="gxela-target-box">

                                <div class="gxela-target-label">
                                    选修目标学时
                                </div>

                                <input
                                    id="gxela-targetElective"
                                    class="gxela-target-input"
                                    type="number"
                                    step="0.25"
                                    min="0"
                                    value="${targetHours.elective || 0}"
                                >

                            </div>

                        </div>

                        <div class="gxela-target-hint">
                            修改后立即保存；下次打开脚本会自动恢复本次的年度与目标学时。
                        </div>


                        <div class="gxela-stat-grid">

                            <div class="gxela-stat">

                                <div
                                    id="gxela-learnedCompulsory"
                                    class="gxela-stat-value"
                                >
                                    0
                                </div>

                                <div class="gxela-stat-label">
                                    已学必修
                                </div>

                            </div>


                            <div class="gxela-stat">

                                <div
                                    id="gxela-learnedElective"
                                    class="gxela-stat-value"
                                >
                                    0
                                </div>

                                <div class="gxela-stat-label">
                                    已学选修
                                </div>

                            </div>


                            <div class="gxela-stat">

                                <div
                                    id="gxela-pendingCompulsory"
                                    class="gxela-stat-value"
                                >
                                    0
                                </div>

                                <div class="gxela-stat-label">
                                    已选必修
                                </div>

                            </div>


                            <div class="gxela-stat">

                                <div
                                    id="gxela-pendingElective"
                                    class="gxela-stat-value"
                                >
                                    0
                                </div>

                                <div class="gxela-stat-label">
                                    已选选修
                                </div>

                            </div>

                        </div>


                        <div class="gxela-remaining">

                            <div class="gxela-remaining-row">

                                <span>
                                    还需选 / 学
                                </span>

                                <strong>
                                    必修
                                    <span id="gxela-remainingComp">
                                        0
                                    </span>
                                    学时
                                </strong>

                            </div>


                            <div class="gxela-remaining-row">

                                <span></span>

                                <strong>
                                    选修
                                    <span id="gxela-remainingElect">
                                        0
                                    </span>
                                    学时
                                </strong>

                            </div>

                        </div>

                    </div>

                </div>


                <!-- 播放状态 -->

                <div class="gxela-card">

                    <div
                        class="gxela-card-title"
                        data-target="playing-card"
                    >

                        <div class="gxela-card-title-left">

                            <span class="gxela-card-icon">
                                ▶️
                            </span>

                            <span>
                                播放状态
                            </span>

                        </div>

                        <span class="gxela-card-arrow">
                            ▼
                        </span>

                    </div>


                    <div
                        id="playing-card"
                        class="gxela-card-content"
                    >

                        <div class="gxela-playing">

                            <div
                                id="gxela-currentCourse"
                                class="gxela-playing-name"
                            >
                                -
                            </div>


                        </div>


                        <div class="gxela-next">

                            下一节：
                            <span id="gxela-nextCourse">
                                -
                            </span>

                        </div>

                    </div>

                </div>


                <!-- 主控制 -->

                <div class="gxela-card">

                    <div
                        class="gxela-card-title"
                        data-target="control-card"
                    >

                        <div class="gxela-card-title-left">

                            <span class="gxela-card-icon">
                                ⚙️
                            </span>

                            <span>
                                自动学习控制
                            </span>

                        </div>

                        <span class="gxela-card-arrow">
                            ▼
                        </span>

                    </div>


                    <div
                        id="control-card"
                        class="gxela-card-content"
                    >

                        <div id="gxela-status">
                            [状态] 等待操作
                        </div>


                        <div class="gxela-buttons">

                            <button
                                id="gxela-startBtn"
                                class="gxela-btn primary"
                            >
                                ▶ 开始自动学习
                            </button>

                            <button
                                id="gxela-stopBtn"
                                class="gxela-btn"
                            >
                                ⏹ 停止
                            </button>

                            <button
                                id="gxela-refreshBtn"
                                class="gxela-btn"
                            >
                                🔄 刷新数据
                            </button>

                            <button
                                id="gxela-resumeBtn"
                                class="gxela-btn"
                            >
                                ↻ 恢复上次学习
                            </button>

                        </div>

                    </div>

                </div>


                <!-- 课程列表 -->

                <div class="gxela-card">

                    <div
                        class="gxela-card-title"
                        data-target="course-card"
                    >

                        <div class="gxela-card-title-left">

                            <span class="gxela-card-icon">
                                📋
                            </span>

                            <span>
                                播放课程列表
                            </span>

                            <span
                                id="gxela-course-count"
                                class="gxela-small"
                            >
                                0 门
                            </span>

                        </div>

                        <span class="gxela-card-arrow">
                            ▼
                        </span>

                    </div>


                    <div
                        id="course-card"
                        class="gxela-card-content"
                    >

                        <div class="gxela-queue-hint">
                            全部待学习课程均显示在此处；列表较长时可在卡片内滚动，点击“播放”可直接跳转。
                        </div>

                        <div class="gxela-queue-controls">
                            <button id="gxela-skipCurrentBtn" class="gxela-btn">⏭ 跳过当前</button>
                            <button id="gxela-clearQueueBtn" class="gxela-btn danger">清空本地队列</button>
                        </div>

                        <div id="gxela-course-list">

                            <div class="gxela-empty">
                                暂无课程
                            </div>

                        </div>

                    </div>

                </div>


                <!-- 班级必修同步 -->

                <div class="gxela-card">
                    <div class="gxela-card-title" data-target="class-sync-card">
                        <div class="gxela-card-title-left">
                            <span class="gxela-card-icon">🏫</span>
                            <span>班级必修同步</span>
                        </div>
                        <span class="gxela-card-arrow">▼</span>
                    </div>
                    <div id="class-sync-card" class="gxela-card-content">
                        <div class="gxela-queue-hint">个人自学完成后，按班级入口逐门刷新完成状态；不会改动个人选课。</div>
                        <div id="gxela-classSyncStatus">[状态] 尚未执行</div>
                        <div class="gxela-buttons">
                            <button id="gxela-classSyncStartBtn" class="gxela-btn primary">🔄 刷新班级必修状态</button>
                            <button id="gxela-classSyncRetryBtn" class="gxela-btn">↻ 仅重试失败项</button>
                            <button id="gxela-classSyncStopBtn" class="gxela-btn danger">停止同步</button>
                        </div>
                        <div id="gxela-classSyncResult" class="gxela-queue-hint" style="margin-top:8px;">暂无同步记录</div>
                    </div>
                </div>

                <!-- clientid -->

                <div class="gxela-card">

                    <div
                        class="gxela-card-title"
                        data-target="client-card"
                    >

                        <div class="gxela-card-title-left">

                            <span class="gxela-card-icon">
                                🔑
                            </span>

                            <span>
                                clientid 设置
                            </span>

                        </div>

                        <span class="gxela-card-arrow">
                            ▶
                        </span>

                    </div>


                    <div
                        id="client-card"
                        class="gxela-card-content collapsed"
                    >

                        <div
                            id="gxela-clientid-status"
                            class="gxela-client-status"
                        >
                            尚未设置 clientid
                        </div>


                        <input
                            id="gxela-clientid-input"
                            class="gxela-client-input"
                            placeholder="请输入 clientid"
                        >


                        <div class="gxela-buttons">

                            <button
                                id="gxela-saveClientBtn"
                                class="gxela-btn primary"
                            >
                                保存
                            </button>

                            <button
                                id="gxela-detectClientBtn"
                                class="gxela-btn"
                            >
                                自动检测
                            </button>

                            <button
                                id="gxela-clearClientBtn"
                                class="gxela-btn danger"
                            >
                                清除
                            </button>

                        </div>

                    </div>

                </div>


                <!-- 管理 -->

                <div class="gxela-card">

                    <div
                        class="gxela-card-title"
                        data-target="manage-card"
                    >

                        <div class="gxela-card-title-left">

                            <span class="gxela-card-icon">
                                🛠
                            </span>

                            <span>
                                课程管理
                            </span>

                        </div>

                        <span class="gxela-card-arrow">
                            ▶
                        </span>

                    </div>


                    <div
                        id="manage-card"
                        class="gxela-card-content collapsed"
                    >

                        <div class="gxela-buttons">

                            <button
                                id="gxela-cancelCompBtn"
                                class="gxela-btn danger"
                            >
                                取消全部必修
                            </button>

                            <button
                                id="gxela-cancelElectBtn"
                                class="gxela-btn danger"
                            >
                                取消全部选修
                            </button>

                        </div>

                    </div>

                </div>


                <!-- 日志 -->

                <div class="gxela-card">

                    <div
                        class="gxela-card-title"
                        data-target="log-card"
                    >

                        <div class="gxela-card-title-left">

                            <span class="gxela-card-icon">
                                📝
                            </span>

                            <span>
                                运行日志
                            </span>

                        </div>

                        <span class="gxela-card-arrow">
                            ▶
                        </span>

                    </div>


                    <div
                        id="log-card"
                        class="gxela-card-content collapsed"
                    >

                        <div class="gxela-log-tools">
                            <select id="gxela-logFilter" aria-label="日志筛选">
                                <option value="all">全部日志</option>
                                <option value="warning">仅警告/错误</option>
                                <option value="success">仅成功/完成</option>
                            </select>
                        </div>

                        <div
                            id="gxela-logbox"
                        ></div>

                        <div
                            class="gxela-buttons"
                            style="margin-top:8px;"
                        >

                            <button
                                id="gxela-exportBtn"
                                class="gxela-btn"
                            >
                                导出日志
                            </button>

                            <button
                                id="gxela-clearLogBtn"
                                class="gxela-btn"
                            >
                                清空日志
                            </button>

                        </div>

                    </div>

                </div>


            </div>
        `;


        document.body.appendChild(panel);


        /* ============================================================
         * 折叠卡片
         * ============================================================ */

        panel
            .querySelectorAll(
                '.gxela-card-title'
            )
            .forEach(title => {

                title.addEventListener(
                    'click',
                    () => {

                        const targetId =
                            title.dataset.target;

                        const content =
                            document.getElementById(
                                targetId
                            );

                        if (!content) {
                            return;
                        }

                        const arrow =
                            title.querySelector(
                                '.gxela-card-arrow'
                            );

                        content.classList.toggle(
                            'collapsed'
                        );

                        if (
                            content.classList.contains(
                                'collapsed'
                            )
                        ) {

                            if (arrow) {
                                arrow.textContent = '▶';
                            }

                        } else {

                            if (arrow) {
                                arrow.textContent = '▼';
                            }

                        }

                    }
                );

            });


        /* ============================================================
         * 主面板折叠
         * ============================================================ */

        document
            .getElementById(
                'gxela-header'
            )
            .addEventListener(
                'click',
                () => {

                    const body =
                        document.getElementById(
                            'gxela-body'
                        );

                    const arrow =
                        document.getElementById(
                            'gxela-header-arrow'
                        );

                    body.classList.toggle(
                        'collapsed'
                    );

                    arrow.textContent =
                        body.classList.contains(
                            'collapsed'
                        )
                            ? '▲'
                            : '▼';

                }
            );


        /* ============================================================
         * clientid
         * ============================================================ */

        const cidInput =
            document.getElementById(
                'gxela-clientid-input'
            );

        const cidStatus =
            document.getElementById(
                'gxela-clientid-status'
            );


        if (clientIdCurrent) {

            cidInput.value =
                clientIdCurrent;

            cidStatus.textContent =
                `已设置 · ${clientIdCurrent}`;

        } else {

            cidInput.value =
                FALLBACK_CLIENT_ID;

            cidStatus.textContent =
                '尚未保存 clientid · 输入框已预填备用值';

        }


        document
            .getElementById(
                'gxela-saveClientBtn'
            )
            .addEventListener(
                'click',
                () => {

                    const value =
                        cidInput.value.trim();

                    if (!value) {

                        alert(
                            '请输入 clientid'
                        );

                        return;
                    }

                    saveClientId(
                        value,
                        'manual'
                    );

                }
            );


        document
            .getElementById(
                'gxela-clearClientBtn'
            )
            .addEventListener(
                'click',
                clearClientId
            );


        document
            .getElementById(
                'gxela-detectClientBtn'
            )
            .addEventListener(
                'click',
                () => {

                    appendLog(
                        '开始手动检测 clientid...'
                    );

                    const found =
                        scanPageForClientId();

                    if (found) {

                        saveClientId(
                            found,
                            'scan'
                        );

                        setStatus(
                            '[状态] clientid 检测成功'
                        );

                        return;
                    }

                    injectClientIdInterceptorToPage();

                    appendLog(
                        '未找到静态 clientid，等待页面请求捕获'
                    );

                }
            );


        /* ============================================================
         * 目标学时
         * ============================================================ */

        const targetC =
            document.getElementById(
                'gxela-targetCompulsory'
            );

        const targetE =
            document.getElementById(
                'gxela-targetElective'
            );

        const yearInput =
            document.getElementById(
                'gxela-year'
            );


        targetC.addEventListener(
            'input',
            () => {

                targetHours.compulsory =
                    parseFloat(
                        targetC.value
                    ) || 0;

                saveState(
                    'gxela_targets',
                    targetHours
                );

                appendLog(
                    `保存必修目标：${targetHours.compulsory}`
                );

                updateRemainingDisplay();

            }
        );


        targetE.addEventListener(
            'input',
            () => {

                targetHours.elective =
                    parseFloat(
                        targetE.value
                    ) || 0;

                saveState(
                    'gxela_targets',
                    targetHours
                );

                appendLog(
                    `保存选修目标：${targetHours.elective}`
                );

                updateRemainingDisplay();

            }
        );


        yearInput.addEventListener(
            'change',
            () => {

                currentYear =
                    parseInt(
                        yearInput.value
                    ) ||
                    new Date().getFullYear();

                saveState(
                    'gxela_year',
                    currentYear
                );

                appendLog(
                    `学习年份设置为 ${currentYear}`
                );

            }
        );


        /* ============================================================
         * 按钮
         * ============================================================ */

        document
            .getElementById(
                'gxela-startBtn'
            )
            .addEventListener(
                'click',
                onStartClicked
            );


        document
            .getElementById(
                'gxela-stopBtn'
            )
            .addEventListener(
                'click',
                onStopClicked
            );


        document
            .getElementById(
                'gxela-resumeBtn'
            )
            .addEventListener(
                'click',
                resumeLastSession
            );


        document
            .getElementById(
                'gxela-refreshBtn'
            )
            .addEventListener(
                'click',
                async () => {

                    setStatus(
                        '正在刷新数据...'
                    );

                    await refreshLearnedAndPending(
                        currentYear
                    );

                    await refreshCourseList();

                    setStatus(
                        '[状态] 数据刷新完成'
                    );

                }
            );


        document
            .getElementById(
                'gxela-cancelCompBtn'
            )
            .addEventListener(
                'click',
                async () => {

                    await cancelSelectedByType(3);

                }
            );


        document
            .getElementById(
                'gxela-cancelElectBtn'
            )
            .addEventListener(
                'click',
                async () => {

                    await cancelSelectedByType(2);

                }
            );

        document.getElementById('gxela-skipCurrentBtn').addEventListener(
            'click',
            skipCurrentCourse
        );

        document.getElementById('gxela-clearQueueBtn').addEventListener(
            'click',
            () => {
                const queue = loadState('g_selectedQueue', []);
                if (!queue.length) return;
                if (!confirm(`清空本地播放队列中的 ${queue.length} 门课程？\n不会取消网站选课，也不会删除学习记录。`)) return;
                saveState('g_selectedQueue', []);
                saveState('g_currentIndex', 0);
                saveState('gxela_isLearning', false);
                persistPlaybackSession('queue-cleared');
                renderCourseList();
                setCurrentNext('-', '-');
                setStatus('[状态] 本地队列已清空');
                appendLog('用户清空本地播放队列');
            }
        );

        document.getElementById('gxela-classSyncStartBtn').addEventListener(
            'click',
            () => startClassSync(false)
        );

        document.getElementById('gxela-classSyncRetryBtn').addEventListener(
            'click',
            () => startClassSync(true)
        );

        document.getElementById('gxela-classSyncStopBtn').addEventListener(
            'click',
            stopClassSync
        );

        document
            .getElementById(
                'gxela-exportBtn'
            )
            .addEventListener(
                'click',
                exportLogs
            );


        document
            .getElementById(
                'gxela-clearLogBtn'
            )
            .addEventListener(
                'click',
                () => {

                    logs = [];

                    saveState(
                        'gxela_logs',
                        []
                    );

                    const box =
                        document.getElementById(
                            'gxela-logbox'
                        );

                    if (box) {
                        box.textContent = '';
                    }

                    appendLog(
                        '日志已清空'
                    );

                }
            );


        const logBox =
            document.getElementById(
                'gxela-logbox'
            );

        if (logBox) {
            document.getElementById('gxela-logFilter').value = logFilter;
            renderLogs(true);
        }

        document.getElementById('gxela-logFilter').addEventListener('change', event => {
            logFilter = event.target.value;
            saveState('gxela_logFilter', logFilter);
            renderLogs(true);
        });


        updateStartButton();

        restorePlaybackDisplay();

        saveClassSyncState();

    }


    /* ================================================================
     * 状态
     * ================================================================ */

    function setStatus(text) {

        const element =
            document.getElementById(
                'gxela-status'
            );

        if (element) {
            element.textContent =
                text;
        }

    }


    /* ================================================================
     * 课程显示
     * ================================================================ */

    function formatCourseLabel(
        item,
        actualSec
    ) {

        if (!item) {
            return '-';
        }

        const name =
            item.name ||
            `lessonId:${item.lessonId}`;

        const classHour =
            Number(item.classHour) || 0;

        const predictedMin =
            item.letime
                ? Number(item.letime)
                : classHour
                    ? Math.round(
                        classHour * 60
                    )
                    : null;

        const predicted =
            predictedMin
                ? `${predictedMin} 分`
                : '-';

        let actual = '';

        if (
            actualSec &&
            !isNaN(actualSec) &&
            actualSec > 0
        ) {

            const minutes =
                Math.floor(
                    actualSec / 60
                );

            const seconds =
                Math.round(
                    actualSec % 60
                );

            actual =
                ` · 实际 ${minutes}分${seconds}秒`;

        }

        return (
            `${name} · ${classHour} 学时 · 预计 ${predicted}${actual}`
        );

    }


    function setCurrentNext(
        current,
        next
    ) {

        const currentElement =
            document.getElementById(
                'gxela-currentCourse'
            );

        const nextElement =
            document.getElementById(
                'gxela-nextCourse'
            );

        if (currentElement) {
            currentElement.textContent =
                current || '-';
        }

        if (nextElement) {
            nextElement.textContent =
                next || '-';
        }

    }


    function restorePlaybackDisplay() {

        try {

            const queue =
                loadState(
                    'g_selectedQueue',
                    []
                );

            const index =
                loadState(
                    'g_currentIndex',
                    0
                ) || 0;

            if (
                Array.isArray(queue) &&
                queue.length
            ) {

                const current =
                    queue[index] ||
                    null;

                const next =
                    queue[index + 1] ||
                    null;

                setCurrentNext(
                    formatCourseLabel(
                        current,
                        null
                    ),
                    formatCourseLabel(
                        next,
                        null
                    )
                );

                renderCourseList();

                const session = loadState('gxela_playbackSession', null);
                if (session && session.updatedAt) {
                    setStatus(`[状态] 已恢复队列：第 ${index + 1}/${queue.length} 门（${new Date(session.updatedAt).toLocaleString()}）`);
                }

            }

        } catch (e) {}

    }


    /* 保存可恢复的播放上下文，供刷新页面或重新打开浏览器后继续使用。 */
    function persistPlaybackSession(reason = '') {

        const queue = loadState('g_selectedQueue', []);
        const index = loadState('g_currentIndex', 0) || 0;
        const current = Array.isArray(queue) ? queue[index] : null;

        saveState('gxela_playbackSession', {
            isLearning: loadState('gxela_isLearning', false),
            currentIndex: index,
            currentLessonId: current ? String(current.lessonId) : null,
            queueLength: Array.isArray(queue) ? queue.length : 0,
            updatedAt: new Date().toISOString(),
            reason
        });

    }


    /* ================================================================
     * 剩余学时
     * ================================================================ */

    function updateRemainingDisplay() {

        const learnedComp =
            parseFloat(
                document.getElementById(
                    'gxela-learnedCompulsory'
                )?.textContent || 0
            ) || 0;

        const learnedElect =
            parseFloat(
                document.getElementById(
                    'gxela-learnedElective'
                )?.textContent || 0
            ) || 0;

        const pendingComp =
            parseFloat(
                document.getElementById(
                    'gxela-pendingCompulsory'
                )?.textContent || 0
            ) || 0;

        const pendingElect =
            parseFloat(
                document.getElementById(
                    'gxela-pendingElective'
                )?.textContent || 0
            ) || 0;


        const remainComp =
            Math.max(
                0,
                targetHours.compulsory -
                learnedComp -
                pendingComp
            );


        const remainElect =
            Math.max(
                0,
                targetHours.elective -
                learnedElect -
                pendingElect
            );


        const comp =
            document.getElementById(
                'gxela-remainingComp'
            );

        const elect =
            document.getElementById(
                'gxela-remainingElect'
            );


        if (comp) {
            comp.textContent =
                remainComp.toFixed(2);
        }

        if (elect) {
            elect.textContent =
                remainElect.toFixed(2);
        }

    }


    /* ================================================================
     * /my
     * ================================================================ */

    async function fetchMyCourses(
        lessonType,
        passStatus,
        year
    ) {

        const all = [];

        let page = 1;

        try {

            while (true) {

                const url =
                    `https://www.gxela.gov.cn/gateway/auth/course/lesson/page/my?pageNum=${page}&pageSize=${PAGE_SIZE}&queryLessonType=${lessonType}&passStatus=${passStatus}&year=${year}`;

                const response =
                    await gmRequestRaw({
                        method: 'GET',
                        url,
                        headers: {
                            platform: 'web',
                            Accept:
                                'application/json, text/plain, */*'
                        }
                    });


                if (
                    response.status !== 200
                ) {

                    appendLog(
                        `/my HTTP ${response.status}`
                    );

                    break;
                }


                let data = {};

                try {

                    data =
                        JSON.parse(
                            response.responseText ||
                            '{}'
                        );

                } catch (e) {}


                if (
                    !data.rows ||
                    !data.rows.length
                ) {
                    break;
                }


                all.push(
                    ...data.rows
                );


                const total =
                    data.total ||
                    all.length;


                appendLog(
                    `/my type=${lessonType} status=${passStatus} page=${page} rows=${data.rows.length} (${all.length}/${total})`
                );


                page++;


                if (
                    all.length >= total
                ) {
                    break;
                }

            }

        } catch (e) {

            appendLog(
                `/my 请求异常：${e}`
            );

        }


        return all;

    }


    /* ================================================================
     * 刷新学时
     * ================================================================ */

    async function refreshLearnedAndPending(
        year
    ) {

        appendLog(
            '刷新已学/已选未完成课程...'
        );


        try {

            const mustDone =
                await fetchMyCourses(
                    3,
                    1,
                    year
                );

            const electDone =
                await fetchMyCourses(
                    2,
                    1,
                    year
                );

            const mustPending =
                await fetchMyCourses(
                    3,
                    2,
                    year
                );

            const electPending =
                await fetchMyCourses(
                    2,
                    2,
                    year
                );


            const learnedComp =
                mustDone.reduce(
                    (sum, item) =>
                        sum +
                        (
                            Number(
                                item.classHour
                            ) || 0
                        ),
                    0
                );


            const learnedElect =
                electDone.reduce(
                    (sum, item) =>
                        sum +
                        (
                            Number(
                                item.classHour
                            ) || 0
                        ),
                    0
                );


            const pendingComp =
                mustPending.reduce(
                    (sum, item) =>
                        sum +
                        (
                            Number(
                                item.classHour
                            ) || 0
                        ),
                    0
                );


            const pendingElect =
                electPending.reduce(
                    (sum, item) =>
                        sum +
                        (
                            Number(
                                item.classHour
                            ) || 0
                        ),
                    0
                );


            document.getElementById(
                'gxela-learnedCompulsory'
            ).textContent =
                learnedComp.toFixed(2);


            document.getElementById(
                'gxela-learnedElective'
            ).textContent =
                learnedElect.toFixed(2);


            document.getElementById(
                'gxela-pendingCompulsory'
            ).textContent =
                pendingComp.toFixed(2);


            document.getElementById(
                'gxela-pendingElective'
            ).textContent =
                pendingElect.toFixed(2);


            updateRemainingDisplay();


            pendingIdsSet =
                new Set();


            mustPending.forEach(
                item =>
                    pendingIdsSet.add(
                        String(
                            item.lessonId
                        )
                    )
            );


            electPending.forEach(
                item =>
                    pendingIdsSet.add(
                        String(
                            item.lessonId
                        )
                    )
            );


            saveState(
                'gxela_pendingIds',
                Array.from(
                    pendingIdsSet
                )
            );


            appendLog(
                `学时刷新完成：必修已学 ${learnedComp}，选修已学 ${learnedElect}`
            );


            return {
                learnedComp,
                learnedElect,
                pendingComp,
                pendingElect
            };


        } catch (e) {

            appendLog(
                '刷新学时失败：' +
                e
            );

            return null;

        }

    }


    /* ================================================================
     * 候选课程
     * ================================================================ */

    async function fetchAllCandidates(
        lessonType,
        year
    ) {

        const all = [];

        let page = 1;

        try {

            while (true) {

                const url =
                    'https://www.gxela.gov.cn/gateway/auth/course/lesson/page';

                const body = {
                    pageNum: page,
                    pageSize: PAGE_SIZE,
                    queryLessonType:
                        lessonType,
                    passStatus: 2,
                    year: year,
                    lessonName: '',
                    teachers: '',
                    teacherPosition: '',
                    teacherUnit: '',
                    idList: [],
                    orderType: '',
                    sortType: ''
                };


                const response =
                    await gmRequestRaw({
                        method: 'POST',
                        url,
                        data:
                            JSON.stringify(body),
                        headers: {
                            'Content-Type':
                                'application/json;charset=UTF-8',
                            Accept:
                                'application/json, text/plain, */*'
                        }
                    });


                if (
                    response.status !== 200
                ) {
                    break;
                }


                let data = {};

                try {
                    data =
                        JSON.parse(
                            response.responseText ||
                            '{}'
                        );
                } catch (e) {}


                if (
                    !data.rows ||
                    !data.rows.length
                ) {
                    break;
                }


                all.push(
                    ...data.rows
                );


                const total =
                    data.total ||
                    all.length;


                appendLog(
                    `候选库 type=${lessonType} page=${page} ${all.length}/${total}`
                );


                page++;


                if (
                    all.length >= total
                ) {
                    break;
                }

            }

        } catch (e) {

            appendLog(
                '候选库请求异常：' +
                e
            );

        }


        return all;

    }


    /* ================================================================
     * 贪心选课
     * ================================================================ */

    function greedySelect(
        candidates,
        target,
        alreadySelectedIds
    ) {

        const filtered =
            candidates.filter(
                item => {

                    const isSelect =
                        item.isSelect === undefined
                            ? 0
                            : Number(
                                item.isSelect
                            );

                    return (
                        isSelect === 0 &&
                        !alreadySelectedIds.has(
                            String(
                                item.lessonId
                            )
                        )
                    );

                }
            );


        filtered.forEach(
            item => {

                const hour =
                    Number(
                        item.classHour
                    ) || 0;

                const minutes =
                    item.letime
                        ? Number(
                            item.letime
                        )
                        : hour * 60;


                item._letimeMin =
                    minutes > 0
                        ? minutes
                        : 1;


                item._eff =
                    hour /
                    item._letimeMin;

            }
        );


        filtered.sort(
            (a, b) => {

                if (
                    b._eff !==
                    a._eff
                ) {
                    return (
                        b._eff -
                        a._eff
                    );
                }

                return (
                    a._letimeMin -
                    b._letimeMin
                );

            }
        );


        const selected = [];

        let accumulated = 0;


        for (
            const item of filtered
        ) {

            if (
                accumulated >=
                target
            ) {
                break;
            }

            selected.push(
                item
            );

            accumulated +=
                Number(
                    item.classHour
                ) || 0;

        }


        return selected;

    }


    /* ================================================================
     * 选课
     * ================================================================ */

    async function selectLessonsBatch(
        list
    ) {

        const success = [];


        for (
            const item of list
        ) {

            let ok = false;


            for (
                let attempt = 0;
                attempt <= SELECT_MAX_RETRIES;
                attempt++
            ) {

                try {

                    const url =
                        `https://www.gxela.gov.cn/gateway/auth/user/study/lesson/selection?lessonId=${item.lessonId}&tcLessonId=0`;


                    const response =
                        await gmRequestRaw({
                            method: 'POST',
                            url,
                            data: '',
                            headers: {
                                Accept:
                                    'application/json, text/plain, */*'
                            }
                        });


                    if (
                        response.status === 200
                    ) {

                        let body = {};

                        try {

                            body =
                                JSON.parse(
                                    response.responseText ||
                                    '{}'
                                );

                        } catch (e) {}


                        appendLog(
                            `选课 ${item.name} → ${body.msg || body.code || ''}`
                        );


                        if (
                            Number(
                                body.code
                            ) === 200
                        ) {

                            ok = true;

                            break;

                        }

                    }

                } catch (e) {

                    appendLog(
                        `选课异常：${item.name} ${e}`
                    );

                }


                if (
                    attempt <
                    SELECT_MAX_RETRIES
                ) {

                    await sleep(
                        800 +
                        attempt * 400
                    );

                }

            }


            if (ok) {

                success.push(
                    item
                );

                pendingIdsSet.add(
                    String(
                        item.lessonId
                    )
                );

                saveState(
                    'gxela_pendingIds',
                    Array.from(
                        pendingIdsSet
                    )
                );

            }


            await sleep(
                SELECT_DELAY_MS
            );

        }


        await refreshLearnedAndPending(
            currentYear
        );


        return success;

    }


    /* ================================================================
     * 取消课程
     * ================================================================ */

    async function cancelLessonsBatch(
        list
    ) {

        const success = [];


        for (
            const item of list
        ) {

            let ok = false;


            for (
                let attempt = 0;
                attempt <= CANCEL_MAX_RETRIES;
                attempt++
            ) {

                try {

                    const url =
                        `https://www.gxela.gov.cn/gateway/auth/user/study/lesson/selection/cancel?lessonId=${item.lessonId}`;


                    const response =
                        await gmRequestRaw({
                            method: 'POST',
                            url,
                            data: '',
                            headers: {
                                Accept:
                                    'application/json, text/plain, */*'
                            }
                        });


                    if (
                        response.status === 200
                    ) {

                        let body = {};

                        try {

                            body =
                                JSON.parse(
                                    response.responseText ||
                                    '{}'
                                );

                        } catch (e) {}


                        appendLog(
                            `取消 ${item.name} → ${body.msg || body.code || ''}`
                        );


                        if (
                            Number(
                                body.code
                            ) === 200
                        ) {

                            ok = true;

                            break;

                        }

                    }

                } catch (e) {

                    appendLog(
                        `取消异常：${item.name} ${e}`
                    );

                }


                if (
                    attempt <
                    CANCEL_MAX_RETRIES
                ) {

                    await sleep(
                        600 +
                        attempt * 400
                    );

                }

            }


            if (ok) {

                success.push(
                    item
                );

                pendingIdsSet.delete(
                    String(
                        item.lessonId
                    )
                );

            }


            await sleep(
                CANCEL_DELAY_MS
            );

        }


        saveState(
            'gxela_pendingIds',
            Array.from(
                pendingIdsSet
            )
        );


        await refreshLearnedAndPending(
            currentYear
        );


        return success;

    }


    async function cancelSelectedByType(
        lessonType
    ) {

        setStatus(
            '正在获取已选课程...'
        );


        const list =
            await fetchMyCourses(
                lessonType,
                2,
                currentYear
            );


        if (!list.length) {

            setStatus(
                '[状态] 没有可取消课程'
            );

            return;

        }

        const typeName = lessonType === 3 ? '必修' : '选修';
        const totalHours = list.reduce(
            (sum, item) => sum + (Number(item.classHour) || 0),
            0
        );
        const preview = list
            .slice(0, 8)
            .map((item, index) => `${index + 1}. ${item.name || item.lessonId}（${Number(item.classHour || 0).toFixed(2)} 学时）`)
            .join('\n');
        const remainder = list.length > 8 ? `\n…另有 ${list.length - 8} 门课程` : '';

        if (!confirm(
            `请确认取消以下 ${typeName} 已选课程：\n\n共 ${list.length} 门，${totalHours.toFixed(2)} 学时\n${preview}${remainder}\n\n此操作会实际取消网站选课，确定继续吗？`
        )) {
            setStatus('[状态] 已取消本次取消课程操作');
            appendLog(`用户取消了批量取消${typeName}课程操作`);
            return;
        }


        appendLog(
            `准备取消 ${list.length} 门课程`
        );


        const success =
            await cancelLessonsBatch(
                list
            );


        setStatus(
            `[状态] 取消完成 ${success.length}/${list.length}`
        );

    }


    /* ================================================================
     * 播放队列
     * ================================================================ */

    async function fetchPlayQueue(
        year
    ) {

        const mustPending =
            await fetchMyCourses(
                3,
                2,
                year
            );


        const electPending =
            await fetchMyCourses(
                2,
                2,
                year
            );


        const all = [
            ...mustPending,
            ...electPending
        ];


        all.sort(
            (a, b) => {

                const typeA =
                    Number(
                        a.lessonType ||
                        (
                            a.isMust
                                ? 3
                                : 2
                        )
                    );

                const typeB =
                    Number(
                        b.lessonType ||
                        (
                            b.isMust
                                ? 3
                                : 2
                        )
                    );


                if (
                    typeA !==
                    typeB
                ) {

                    return (
                        typeB -
                        typeA
                    );

                }


                const progressA =
                    Number(
                        a.learnProgress
                    ) || 0;

                const progressB =
                    Number(
                        b.learnProgress
                    ) || 0;


                if (
                    progressA !==
                    progressB
                ) {

                    return (
                        progressB -
                        progressA
                    );

                }


                const remainA =
                    (
                        Number(
                            a.classHour
                        ) || 0
                    ) *
                    (
                        1 -
                        progressA / 100
                    );


                const remainB =
                    (
                        Number(
                            b.classHour
                        ) || 0
                    ) *
                    (
                        1 -
                        progressB / 100
                    );


                return (
                    remainA -
                    remainB
                );

            }
        );


        const seen =
            new Set();

        const result = [];


        for (
            const item of all
        ) {

            const id =
                String(
                    item.lessonId
                );

            if (
                seen.has(id)
            ) {
                continue;
            }

            seen.add(id);

            result.push(item);

        }


        saveState(
            'g_selectedQueue',
            result
        );

        saveState(
            'g_currentIndex',
            0
        );

        persistPlaybackSession('queue-generated');


        return result;

    }


    /* ================================================================
     * 课程列表 UI
     * ================================================================ */

    function renderCourseList() {

        const listElement =
            document.getElementById(
                'gxela-course-list'
            );

        const countElement =
            document.getElementById(
                'gxela-course-count'
            );


        if (!listElement) {
            return;
        }


        const queue =
            loadState(
                'g_selectedQueue',
                []
            );


        const currentIndex =
            loadState(
                'g_currentIndex',
                0
            ) || 0;


        if (
            countElement
        ) {

            countElement.textContent =
                `${queue.length} 门`;

        }


        if (
            !Array.isArray(queue) ||
            !queue.length
        ) {

            listElement.innerHTML =
                `
                    <div class="gxela-empty">
                        暂无待学习课程
                    </div>
                `;

            return;

        }


        listElement.innerHTML =
            queue
                .map(
                    (
                        item,
                        index
                    ) => {

                        const type =
                            Number(
                                item.lessonType ||
                                (
                                    item.isMust
                                        ? 3
                                        : 2
                                )
                            );


                        const typeText =
                            type === 3
                                ? '必修'
                                : '选修';


                        const progress =
                            Number(
                                item.learnProgress
                            ) || 0;


                        const current =
                            index ===
                            currentIndex;


                        return `

                            <div
                                class="gxela-course-item ${current ? 'gxela-course-current' : ''}"
                            >

                                <div
                                    class="gxela-course-index"
                                >
                                    ${index + 1}
                                </div>


                                <div
                                    class="gxela-course-main"
                                >

                                    <div
                                        class="gxela-course-name"
                                        title="${escapeHtml(item.name || '')}"
                                    >
                                        ${escapeHtml(item.name || `课程 ${item.lessonId}`)}
                                    </div>


                                    <div
                                        class="gxela-course-meta"
                                    >
                                        ${typeText}
                                        ·
                                        ${Number(item.classHour || 0).toFixed(2)}
                                        学时
                                        ·
                                        进度 ${progress.toFixed(1)}%
                                    </div>

                                </div>


                                <div class="gxela-course-actions">
                                    <button class="gxela-course-action gxela-course-top" data-lesson-id="${item.lessonId}" title="移到播放列表顶部">置顶</button>
                                    <button class="gxela-course-action gxela-course-remove" data-lesson-id="${item.lessonId}" title="仅从本地播放列表移除，不会取消选课">移出</button>
                                    <button
                                        class="gxela-course-play"
                                        data-lesson-id="${item.lessonId}"
                                    >
                                        ▶ 播放
                                    </button>
                                </div>

                            </div>

                        `;

                    }
                )
                .join('');


        listElement
            .querySelectorAll(
                '.gxela-course-play'
            )
            .forEach(
                button => {

                    button.addEventListener(
                        'click',
                        event => {

                            event.stopPropagation();

                            const lessonId =
                                button.dataset.lessonId;

                            playSpecificCourse(
                                lessonId
                            );

                        }
                    );

                }
            );

        listElement.querySelectorAll('.gxela-course-top').forEach(button => {
            button.addEventListener('click', event => {
                event.stopPropagation();
                moveCourseToTop(button.dataset.lessonId);
            });
        });

        listElement.querySelectorAll('.gxela-course-remove').forEach(button => {
            button.addEventListener('click', event => {
                event.stopPropagation();
                removeCourseFromQueue(button.dataset.lessonId);
            });
        });

    }


    function escapeHtml(
        text
    ) {

        return String(text)
            .replace(
                /&/g,
                '&amp;'
            )
            .replace(
                /</g,
                '&lt;'
            )
            .replace(
                />/g,
                '&gt;'
            )
            .replace(
                /"/g,
                '&quot;'
            )
            .replace(
                /'/g,
                '&#039;'
            );

    }


    /* ================================================================
     * 播放队列操作（仅修改本地队列，不影响网站上的选课记录）
     * ================================================================ */

    function moveCourseToTop(lessonId) {

        const queue = loadState('g_selectedQueue', []);
        const index = queue.findIndex(item => String(item.lessonId) === String(lessonId));

        if (index < 0) return;

        const item = queue.splice(index, 1)[0];
        queue.unshift(item);
        saveState('g_selectedQueue', queue);
        saveState('g_currentIndex', 0);
        persistPlaybackSession('course-moved-to-top');
        renderCourseList();
        restorePlaybackDisplay();
        appendLog(`已将课程置顶：${item.name || item.lessonId}`);

    }


    function removeCourseFromQueue(lessonId) {

        const queue = loadState('g_selectedQueue', []);
        const index = queue.findIndex(item => String(item.lessonId) === String(lessonId));

        if (index < 0) return;

        const item = queue[index];

        if (!confirm(`仅从本地播放列表移除“${item.name || item.lessonId}”？\n这不会取消网站中的选课。`)) return;

        queue.splice(index, 1);
        const currentIndex = Math.min(loadState('g_currentIndex', 0) || 0, Math.max(0, queue.length - 1));
        saveState('g_selectedQueue', queue);
        saveState('g_currentIndex', currentIndex);
        persistPlaybackSession('course-removed-from-queue');
        renderCourseList();
        restorePlaybackDisplay();
        appendLog(`已从本地队列移除：${item.name || item.lessonId}`);

    }


    function skipCurrentCourse() {

        const queue = loadState('g_selectedQueue', []);
        const index = loadState('g_currentIndex', 0) || 0;
        const item = queue[index];

        if (!item) {
            setStatus('[状态] 没有可跳过的课程');
            return;
        }

        if (!confirm(`跳过“${item.name || item.lessonId}”？\n课程仍保留在网站已选列表中，仅从本地播放队列移除。`)) return;

        queue.splice(index, 1);
        saveState('g_selectedQueue', queue);
        saveState('g_currentIndex', Math.min(index, Math.max(0, queue.length - 1)));
        persistPlaybackSession('current-course-skipped');
        renderCourseList();
        appendLog(`已跳过当前课程：${item.name || item.lessonId}`);

        if (loadState('gxela_isLearning', false)) autoPlayQueue();

    }


    /* ================================================================
     * 手动播放某课程
     * ================================================================ */

    function playSpecificCourse(
        lessonId
    ) {

        const queue =
            loadState(
                'g_selectedQueue',
                []
            );


        const index =
            queue.findIndex(
                item =>
                    String(
                        item.lessonId
                    ) ===
                    String(
                        lessonId
                    )
            );


        if (index >= 0) {

            saveState(
                'g_currentIndex',
                index
            );

        }


        saveState(
            'gxela_isLearning',
            true
        );

        persistPlaybackSession('manual-play-selected');


        appendLog(
            `手动播放课程 lessonId=${lessonId}`
        );


        jumpToLesson(
            lessonId
        );

    }


    /* ================================================================
     * 跳转播放
     * ================================================================ */

    function jumpToLesson(
        lessonId
    ) {

        const url =
            `https://www.gxela.gov.cn/watch?lessonId=${lessonId}&tcLessonId=0&lessonOrigin=selflearn`;


        window.location.href =
            url;

    }


    function autoPlayQueue() {

        if (
            !loadState(
                'gxela_isLearning',
                false
            )
        ) {

            setStatus(
                '[状态] 已停止'
            );

            return;

        }


        const queue =
            loadState(
                'g_selectedQueue',
                []
            );


        const index =
            loadState(
                'g_currentIndex',
                0
            ) || 0;


        if (
            !Array.isArray(queue) ||
            !queue.length
        ) {

            setStatus(
                '[状态] 队列为空'
            );

            saveState(
                'gxela_isLearning',
                false
            );

            return;

        }


        if (
            index >= queue.length
        ) {

            appendLog(
                '🎉 所有课程播放完成'
            );


            saveState(
                'gxela_isLearning',
                false
            );


            setStatus(
                '[状态] 全部完成'
            );

            appendLog('个人学习队列完成，准备自动检查班级必修同步任务');
            setTimeout(() => startClassSync(false), 1000);


            return;

        }


        const current =
            queue[index];

        const next =
            queue[index + 1] ||
            null;


        setCurrentNext(
            formatCourseLabel(
                current,
                null
            ),
            formatCourseLabel(
                next,
                null
            )
        );


        renderCourseList();

        persistPlaybackSession('auto-play-next');


        appendLog(
            `跳转播放 ${index + 1}/${queue.length}：${current.name || current.lessonId}`
        );


        jumpToLesson(
            current.lessonId
        );

    }


    function resumeLastSession() {

        const session = loadState('gxela_playbackSession', null);
        const queue = loadState('g_selectedQueue', []);

        if (!session || !Array.isArray(queue) || !queue.length) {
            setStatus('[状态] 没有可恢复的播放队列');
            return;
        }

        const savedIndex = Math.min(
            Math.max(0, Number(session.currentIndex) || 0),
            queue.length - 1
        );

        saveState('g_currentIndex', savedIndex);
        saveState('gxela_isLearning', true);
        persistPlaybackSession('user-resumed-session');
        appendLog(`恢复上次学习：第 ${savedIndex + 1}/${queue.length} 门`);
        setStatus('[状态] 正在恢复上次学习');
        autoPlayQueue();

    }


    /* ================================================================
     * 开始
     * ================================================================ */

    async function onStartClicked() {

        if (!clientIdCurrent) {

            alert(
                '请先设置 clientid'
            );

            return;

        }


        try {

            currentYear =
                parseInt(
                    document.getElementById(
                        'gxela-year'
                    ).value
                ) ||
                new Date().getFullYear();


            targetHours.compulsory =
                parseFloat(
                    document.getElementById(
                        'gxela-targetCompulsory'
                    ).value
                ) || 0;


            targetHours.elective =
                parseFloat(
                    document.getElementById(
                        'gxela-targetElective'
                    ).value
                ) || 0;


            saveState(
                'gxela_targets',
                targetHours
            );


            saveState(
                'gxela_year',
                currentYear
            );


            saveState(
                'gxela_isLearning',
                true
            );


            setStatus(
                '正在初始化...'
            );


            const data =
                await refreshLearnedAndPending(
                    currentYear
                );


            if (!data) {

                saveState(
                    'gxela_isLearning',
                    false
                );

                setStatus(
                    '[状态] 初始化失败'
                );

                return;

            }


            const remainingComp =
                Math.max(
                    0,
                    targetHours.compulsory -
                    data.learnedComp -
                    data.pendingComp
                );


            const remainingElect =
                Math.max(
                    0,
                    targetHours.elective -
                    data.learnedElect -
                    data.pendingElect
                );


            if (
                remainingComp > 0
            ) {

                setStatus(
                    '正在选择必修课程...'
                );


                const candidates =
                    await fetchAllCandidates(
                        3,
                        currentYear
                    );


                const selected =
                    greedySelect(
                        candidates,
                        remainingComp,
                        pendingIdsSet
                    );


                appendLog(
                    `必修拟选 ${selected.length} 门`
                );


                await selectLessonsBatch(
                    selected
                );

            }


            if (
                remainingElect > 0
            ) {

                setStatus(
                    '正在选择选修课程...'
                );


                const candidates =
                    await fetchAllCandidates(
                        2,
                        currentYear
                    );


                const selected =
                    greedySelect(
                        candidates,
                        remainingElect,
                        pendingIdsSet
                    );


                appendLog(
                    `选修拟选 ${selected.length} 门`
                );


                await selectLessonsBatch(
                    selected
                );

            }


            setStatus(
                '正在生成播放队列...'
            );


            const queue =
                await fetchPlayQueue(
                    currentYear
                );


            renderCourseList();


            if (!queue.length) {

                saveState(
                    'gxela_isLearning',
                    false
                );

                setStatus(
                    '[状态] 没有待学习课程'
                );

                return;

            }


            setStatus(
                `准备播放，共 ${queue.length} 门`
            );


            autoPlayQueue();


        } catch (e) {

            appendLog(
                '开始流程异常：' +
                e
            );

            setStatus(
                '[状态] 异常'
            );

        }

    }


    /* ================================================================
     * 停止
     * ================================================================ */

    function onStopClicked() {

        saveState(
            'gxela_isLearning',
            false
        );

        persistPlaybackSession('user-stopped');


        appendLog(
            '用户停止自动学习'
        );


        setStatus(
            '[状态] 已停止'
        );

    }


    /* ================================================================
     * 页面点击
     *
     * 播放后点击空白区域：
     * 用于消除网站可能出现的 MessageBox
     * ================================================================ */

    function clickBlankArea() {

        try {

            const panel =
                document.getElementById(
                    'gxela-panel'
                );


            const x =
                Math.max(
                    5,
                    Math.floor(
                        window.innerWidth *
                        0.45
                    )
                );


            const y =
                Math.max(
                    5,
                    Math.floor(
                        window.innerHeight *
                        0.08
                    )
                );


            /*
             * 不点击 GXELA 面板，
             * 尽量选择页面空白区域。
             */

            if (
                panel &&
                x >= panel.offsetLeft &&
                x <=
                    panel.offsetLeft +
                    panel.offsetWidth
            ) {

                document.body.click();

            } else {

                const element =
                    document.elementFromPoint(
                        x,
                        y
                    );

                if (element) {

                    element.dispatchEvent(
                        new MouseEvent(
                            'click',
                            {
                                bubbles: true,
                                cancelable: true,
                                view: window
                            }
                        )
                    );

                }

            }


            appendLog(
                '已执行一次页面空白区域点击'
            );


        } catch (e) {

            console.warn(
                'blank click error',
                e
            );

        }

    }


    function scheduleBlankClick() {

        clearTimeout(
            pageClickTimer
        );


        pageClickTimer =
            setTimeout(
                () => {

                    clickBlankArea();

                },
                DIALOG_DISMISS_DELAY
            );

    }


    /* ================================================================
     * 永久静音
     * ================================================================ */

    function forceMute(
        video
    ) {

        if (!video) {
            return;
        }


        try {

            video.muted = true;

            video.volume = 0;

        } catch (e) {}


        /*
         * 防止网站再次修改 muted / volume
         */

        if (
            !video.__gxelaMuteBound
        ) {

            video.__gxelaMuteBound =
                true;


            video.addEventListener(
                'volumechange',
                () => {

                    try {

                        if (
                            !video.muted
                        ) {

                            video.muted =
                                true;

                        }

                        if (
                            video.volume !== 0
                        ) {

                            video.volume = 0;

                        }

                    } catch (e) {}

                }
            );

        }

    }


    /* ================================================================
     * 主动播放
     * ================================================================ */

    async function forcePlayVideo(
        video
    ) {

        if (!video) {
            return false;
        }


        currentVideo =
            video;


        forceMute(
            video
        );


        /*
         * 如果视频已经在播放，
         * 不重复调用 play。
         */

        if (
            !video.paused &&
            !video.ended
        ) {

            appendLog(
                `🎬 video 当前正在播放，保持播放。currentTime=${video.currentTime.toFixed(2)}`
            );

            videoPlayRetryCount = 0;

            // V3.6：即使网站已先行播放，也执行一次空白点击以关闭可能的遮罩/对话框。
            scheduleBlankClick();

            return true;

        }


        /*
         * paused 时主动尝试播放。
         */

        try {

            appendLog(
                `▶ video 当前 paused，主动尝试播放。currentTime=${Number(video.currentTime || 0).toFixed(2)}`
            );


            const result =
                video.play();


            if (
                result &&
                typeof result.then ===
                    'function'
            ) {

                await result;

            }


            forceMute(
                video
            );


            videoPlayRetryCount = 0;


            appendLog(
                '▶ video.play() 成功'
            );


            scheduleBlankClick();


            return true;


        } catch (e) {

            videoPlayRetryCount++;


            appendLog(
                `⚠ video.play() 失败 (${videoPlayRetryCount}/${PLAY_MAX_RETRIES})：${e}`
            );


            if (
                videoPlayRetryCount <
                PLAY_MAX_RETRIES
            ) {

                setTimeout(
                    () => {

                        if (
                            loadState(
                                'gxela_isLearning',
                                false
                            )
                        ) {

                            forcePlayVideo(
                                video
                            );

                        }

                    },
                    PLAY_RETRY_INTERVAL
                );

            }


            return false;

        }

    }


    /* ================================================================
     * 视频查找
     * ================================================================ */

    function findBestVideo() {

        const videos =
            Array.from(
                document.querySelectorAll(
                    'video'
                )
            );


        if (!videos.length) {
            return null;
        }


        /*
         * 优先选择有 src / currentSrc 的 video
         */

        videos.sort(
            (a, b) => {

                const score =
                    video => {

                        let value = 0;

                        if (
                            video.currentSrc
                        ) {
                            value += 10;
                        }

                        if (
                            video.src
                        ) {
                            value += 5;
                        }

                        if (
                            video.duration &&
                            isFinite(
                                video.duration
                            )
                        ) {
                            value += 5;
                        }

                        if (
                            !video.paused
                        ) {
                            value += 20;
                        }

                        return value;

                    };


                return (
                    score(b) -
                    score(a)
                );

            }
        );


        return videos[0];

    }


    /* 轮询后台状态，只有确认课程完成（或已不在未完成列表）才允许切换下一节。 */
    async function waitForLessonCompletion(lessonId) {

        const id = String(lessonId || '');

        for (let attempt = 1; attempt <= 6; attempt++) {
            const completed = [
                ...(await fetchMyCourses(3, 1, currentYear)),
                ...(await fetchMyCourses(2, 1, currentYear))
            ];

            if (completed.some(item => String(item.lessonId) === id)) return true;

            const pending = [
                ...(await fetchMyCourses(3, 2, currentYear)),
                ...(await fetchMyCourses(2, 2, currentYear))
            ];

            const stillPending = pending.some(item => String(item.lessonId) === id);
            setStatus(`[状态] 等待平台确认完成（${attempt}/6）`);
            appendLog(
                stillPending
                    ? `课程尚未同步为完成，${attempt}/6 次确认后重试`
                    : `未在完成列表找到课程，继续确认以避免网络异常误切换（${attempt}/6）`
            );
            await sleep(5000);
        }

        return false;

    }


    /* ================================================================
     * 视频完成
     * ================================================================ */

    async function handleVideoEnded(
        video
    ) {

        appendLog(
            '✅ 当前课程 video ended'
        );

        const endedLessonId =
            new URL(window.location.href)
                .searchParams
                .get('lessonId');

        const confirmed = await waitForLessonCompletion(endedLessonId);

        if (!confirmed) {
            saveState('gxela_isLearning', false);
            persistPlaybackSession('completion-not-confirmed');
            setStatus('[状态] 未确认完成，已停止切换；请稍后刷新后重试');
            appendLog('❌ 未获得平台完成确认，未切换下一门课程');
            return;
        }


        /*
         * 这里非常重要：
         *
         * 不提前切换课程。
         *
         * 只有 video 真正 ended 后，
         * 才推进队列。
         */

        try {

            await refreshLearnedAndPending(
                currentYear
            );

            /*
             * V3.6：每门课程结束后重新从服务器生成队列。
             * 这样已完成课程会消失，下一门的进度、学时与列表高亮均是最新数据。
             */
            const refreshedQueue =
                await fetchPlayQueue(
                    currentYear
                );

            // 后端进度有时会延迟写入；本页已经触发 ended 的课程不应再次播放。
            const nextQueue =
                refreshedQueue.filter(
                    item =>
                        String(item.lessonId) !==
                        String(endedLessonId)
                );

            saveState(
                'g_selectedQueue',
                nextQueue
            );

            saveState(
                'g_currentIndex',
                0
            );

            persistPlaybackSession('completed-course-queue-refreshed');

            renderCourseList();

            if (!nextQueue.length) {

                saveState(
                    'gxela_isLearning',
                    false
                );

                setStatus(
                    '[状态] 全部课程完成'
                );

                setCurrentNext(
                    '-',
                    '-'
                );

                appendLog(
                    '🎉 播放列表已刷新，所有课程均已完成'
                );

                return;

            }

            appendLog(
                `播放列表已刷新，剩余 ${nextQueue.length} 门；即将进入下一门`
            );

            setTimeout(
                autoPlayQueue,
                800
            );

            return;

        } catch (e) {

            appendLog(
                '结束后刷新播放列表异常：' +
                e
            );

            return;

        }


        const queue =
            loadState(
                'g_selectedQueue',
                []
            );


        let index =
            loadState(
                'g_currentIndex',
                0
            ) || 0;


        index++;


        saveState(
            'g_currentIndex',
            index
        );


        renderCourseList();


        appendLog(
            `当前课程完成，队列进入第 ${index + 1}/${queue.length} 门`
        );


        if (
            index >= queue.length
        ) {

            saveState(
                'gxela_isLearning',
                false
            );


            setStatus(
                '[状态] 全部课程完成'
            );


            setCurrentNext(
                '-',
                '-'
            );


            return;

        }


        setTimeout(
            () => {

                autoPlayQueue();

            },
            800
        );

    }


    /* ================================================================
     * 视频卡顿监控
     * ================================================================ */

    function startVideoStuckMonitor(
        video
    ) {

        if (
            video.__gxelaMonitor
        ) {
            return;
        }


        video.__gxelaMonitor =
            true;


        let lastTime =
            Number(
                video.currentTime
            ) || 0;


        let stuckCount = 0;


        const lessonId =
            new URL(
                window.location.href
            )
                .searchParams
                .get(
                    'lessonId'
                ) || '';


        const reloadKey =
            `gxela_reloadAttempts_${lessonId}`;


        const interval =
            setInterval(
                async () => {

                    try {

                        if (
                            !loadState(
                                'gxela_isLearning',
                                false
                            )
                        ) {

                            clearInterval(
                                interval
                            );

                            return;

                        }


                        forceMute(
                            video
                        );


                        if (
                            video.ended
                        ) {

                            clearInterval(
                                interval
                            );

                            return;

                        }


                        /*
                         * 视频正在播放：
                         * 检查 currentTime 是否增长。
                         */

                        if (
                            !video.paused
                        ) {

                            const current =
                                Number(
                                    video.currentTime
                                ) || 0;


                            if (
                                current >
                                lastTime +
                                0.5
                            ) {

                                stuckCount = 0;

                                lastTime =
                                    current;

                                return;

                            }


                            stuckCount++;


                            appendLog(
                                `⚠ 播放疑似卡顿 count=${stuckCount} currentTime=${current.toFixed(2)}`
                            );


                            if (
                                stuckCount >=
                                VIDEO_STUCK_THRESHOLD
                            ) {

                                appendLog(
                                    '尝试恢复视频播放'
                                );


                                await forcePlayVideo(
                                    video
                                );


                                await sleep(
                                    3000
                                );


                                const current2 =
                                    Number(
                                        video.currentTime
                                    ) || 0;


                                if (
                                    current2 >
                                    lastTime +
                                    0.5
                                ) {

                                    appendLog(
                                        '▶ 视频已恢复'
                                    );

                                    stuckCount = 0;

                                    lastTime =
                                        current2;

                                    return;

                                }


                                let reloadAttempts =
                                    parseInt(
                                        loadState(
                                            reloadKey,
                                            0
                                        )
                                    ) || 0;


                                if (
                                    reloadAttempts <
                                    VIDEO_MAX_RELOADS
                                ) {

                                    reloadAttempts++;


                                    saveState(
                                        reloadKey,
                                        reloadAttempts
                                    );


                                    appendLog(
                                        `页面重载恢复，第 ${reloadAttempts}/${VIDEO_MAX_RELOADS} 次`
                                    );


                                    location.reload();

                                    return;

                                }


                                /*
                                 * 达到页面重载上限：
                                 * 跳过当前课程。
                                 */

                                appendLog(
                                    '重载次数达到上限，跳过当前课程'
                                );


                                clearInterval(
                                    interval
                                );


                                const index =
                                    (
                                        loadState(
                                            'g_currentIndex',
                                            0
                                        ) || 0
                                    ) + 1;


                                saveState(
                                    'g_currentIndex',
                                    index
                                );


                                saveState(
                                    reloadKey,
                                    0
                                );


                                setTimeout(
                                    autoPlayQueue,
                                    500
                                );

                            }

                        }

                    } catch (e) {

                        console.warn(
                            'GXELA stuck monitor:',
                            e
                        );

                    }

                },
                VIDEO_STUCK_CHECK_MS
            );

    }


    /* ================================================================
     * 焦点恢复
     *
     * 网站切换焦点时可能：
     *
     * blur
     * ↓
     * 网站暂停 video
     * ↓
     * 网站自己的逻辑再次 play
     *
     * 我们不在 blur 时强行操作，
     * 而是在 pause / focus 时检查并恢复。
     * ================================================================ */

    function bindFocusRecovery(
        video
    ) {

        if (
            video.__gxelaFocusBound
        ) {
            return;
        }


        video.__gxelaFocusBound =
            true;


        video.addEventListener(
            'pause',
            () => {

                if (
                    !loadState(
                        'gxela_isLearning',
                        false
                    )
                ) {
                    return;
                }


                /*
                 * ended 是正常结束，
                 * 不处理。
                 */

                if (
                    video.ended
                ) {
                    return;
                }


                appendLog(
                    `⏸ video 触发 pause，currentTime=${Number(video.currentTime || 0).toFixed(2)}`
                );


                /*
                 * 不在 pause 事件瞬间反复 play。
                 *
                 * 给网站自己的焦点切换逻辑
                 * 一个短暂的处理时间。
                 */

                setTimeout(
                    () => {

                        if (
                            !video.ended &&
                            video.paused &&
                            loadState(
                                'gxela_isLearning',
                                false
                            )
                        ) {

                            appendLog(
                                '⚠ video 仍处于暂停，主动恢复播放'
                            );


                            forcePlayVideo(
                                video
                            );

                        }

                    },
                    BLUR_RECOVER_DELAY
                );

            }
        );


        document.addEventListener(
            'visibilitychange',
            () => {

                if (
                    document.visibilityState !==
                    'visible'
                ) {
                    return;
                }


                if (
                    !loadState(
                        'gxela_isLearning',
                        false
                    )
                ) {
                    return;
                }


                setTimeout(
                    () => {

                        if (
                            !video.ended &&
                            video.paused
                        ) {

                            appendLog(
                                '👁 页面恢复可见，video 仍暂停，恢复播放'
                            );


                            forcePlayVideo(
                                video
                            );

                        }

                    },
                    BLUR_RECOVER_DELAY
                );

            }
        );


        window.addEventListener(
            'focus',
            () => {

                if (
                    !loadState(
                        'gxela_isLearning',
                        false
                    )
                ) {
                    return;
                }


                setTimeout(
                    () => {

                        if (
                            video &&
                            !video.ended &&
                            video.paused
                        ) {

                            appendLog(
                                '🖥 窗口重新获得焦点，恢复视频播放'
                            );


                            forcePlayVideo(
                                video
                            );

                        }

                    },
                    BLUR_RECOVER_DELAY
                );

            }
        );

    }


    /* ================================================================
     * video 初始化
     * ================================================================ */

    async function setupVideo(
        video
    ) {

        if (!video) {
            return;
        }


        currentVideo =
            video;


        forceMute(
            video
        );


        /*
         * 设置 ended
         */

        if (
            !video.__gxelaEndedBound
        ) {

            video.__gxelaEndedBound =
                true;


            video.addEventListener(
                'ended',
                () => {

                    handleVideoEnded(
                        video
                    );

                }
            );

        }


        /*
         * 焦点恢复
         */

        bindFocusRecovery(
            video
        );


        /*
         * 视频加载后再次静音
         */

        video.addEventListener(
            'loadedmetadata',
            () => {

                forceMute(
                    video
                );

            }
        );


        video.addEventListener(
            'canplay',
            () => {

                forceMute(
                    video
                );

            }
        );


        /*
         * 播放状态
         */

        const duration =
            Number(
                video.duration
            );


        const queue =
            loadState(
                'g_selectedQueue',
                []
            );


        const index =
            loadState(
                'g_currentIndex',
                0
            ) || 0;


        const current =
            queue[index] ||
            null;


        const next =
            queue[index + 1] ||
            null;


        setCurrentNext(
            formatCourseLabel(
                current,
                isFinite(duration) &&
                duration > 0
                    ? duration
                    : null
            ),
            formatCourseLabel(
                next,
                null
            )
        );


        setStatus(
            `[状态] 当前课程播放中`
        );


        startVideoStuckMonitor(
            video
        );


        /*
         * 核心：
         * 无论网站是否自动播放，
         * 首次进入发现 video 后，
         * 都主动尝试 play。
         */

        await forcePlayVideo(
            video
        );


        scheduleBlankClick();

    }


    /* ================================================================
     * 班级必修状态同步
     * ================================================================ */

    function saveClassSyncState() {
        saveState('gxela_classSyncState', classSyncState);
        const status = document.getElementById('gxela-classSyncStatus');
        const result = document.getElementById('gxela-classSyncResult');
        if (status) status.textContent = classSyncState.active
            ? `[状态] ${classSyncState.phase === 'quick' ? '快速同步' : '自然播放兜底'}：剩余 ${classSyncState.pending.length} 门`
            : '[状态] 未在同步';
        if (result) result.textContent = `已成功 ${classSyncState.results.length} 门；待处理/失败 ${classSyncState.failed.length} 门`;
    }

    async function classApi(url, method = 'GET', data = null) {
        const response = await gmRequestRaw({
            method,
            url: `https://www.gxela.gov.cn${url}`,
            data: data ? JSON.stringify(data) : undefined,
            headers: {
                platform: 'web',
                Accept: 'application/json, text/plain, */*',
                ...(data ? { 'Content-Type': 'application/json;charset=UTF-8' } : {})
            }
        });
        if (response.status !== 200) throw new Error(`班级接口 HTTP ${response.status}`);
        const body = JSON.parse(response.responseText || '{}');
        if (Number(body.code) !== 200) throw new Error(body.msg || '班级接口返回失败');
        return body;
    }

    async function fetchAllOpenTrainClasses() {
        const all = [];
        for (let page = 1; ; page++) {
            const body = await classApi(`/gateway/auth/trainclass/myTrainclasspage?pageNum=${page}&pageSize=100&trainQueryStatus=1`);
            const rows = body.rows || [];
            all.push(...rows);
            if (!rows.length || all.length >= Number(body.total || 0)) break;
        }
        return all;
    }

    async function fetchClassCategories(trainClassId) {
        const body = await classApi(`/gateway/auth/trainclass/top/${trainClassId}`);
        return (body.data || []).filter(item => item.isLeaf);
    }

    async function fetchClassLessons(trainClassId, categoryId) {
        const all = [];
        for (let page = 1; ; page++) {
            const body = await classApi('/gateway/auth/trainclass/category/lessonpage', 'POST', {
                trainClassId: String(trainClassId), isMust: 1, pageNum: page,
                pageSize: PAGE_SIZE, categoryIds: [String(categoryId)]
            });
            const rows = body.rows || [];
            all.push(...rows);
            if (!rows.length || all.length >= Number(body.total || 0)) break;
        }
        return all;
    }

    async function buildClassSyncTasks() {
        const personalDone = await fetchMyCourses(3, 1, currentYear);
        const doneIds = new Set(personalDone.map(item => String(item.lessonId)));
        const classes = await fetchAllOpenTrainClasses();
        const tasks = [];
        for (const trainClass of classes) {
            const categories = await fetchClassCategories(trainClass.trainClassId);
            for (const category of categories) {
                const lessons = await fetchClassLessons(trainClass.trainClassId, category.categoryId);
                for (const lesson of lessons) {
                    if (doneIds.has(String(lesson.lessonId)) && Number(lesson.passed) !== 1) {
                        tasks.push({ ...lesson, trainClassName: trainClass.name, categoryName: category.name });
                    }
                }
            }
        }
        const seen = new Set();
        return tasks.filter(item => {
            const key = `${item.trainClassId}:${item.tcLessonId}`;
            if (seen.has(key)) return false;
            seen.add(key);
            return true;
        });
    }

    function classWatchUrl(task) {
        return `https://www.gxela.gov.cn/watch?lessonId=${task.lessonId}&tcLessonId=${task.tcLessonId}&lessonOrigin=trainclass`;
    }

    async function startClassSync(retryOnly) {
        try {
            let tasks;
            if (retryOnly) {
                tasks = classSyncState.failed || [];
                if (!tasks.length) {
                    setStatus('[状态] 没有失败项可重试');
                    return;
                }
            } else {
                setStatus('正在读取班级必修课程...');
                tasks = await buildClassSyncTasks();
            }
            classSyncState = { active: true, phase: 'quick', pending: tasks, failed: [], current: null, results: [] };
            appendLog(`班级同步任务已生成：${tasks.length} 门`);
            saveClassSyncState();
            processNextClassTask();
        } catch (e) {
            appendLog(`班级同步初始化失败：${e}`);
            setStatus('[状态] 班级同步初始化失败');
        }
    }

    function stopClassSync() {
        classSyncState.active = false;
        classSyncState.current = null;
        saveClassSyncState();
        appendLog('用户停止班级必修同步');
    }

    function processNextClassTask() {
        if (!classSyncState.active) return;
        if (!classSyncState.pending.length) {
            if (classSyncState.phase === 'quick' && classSyncState.failed.length) {
                classSyncState.phase = 'fallback';
                classSyncState.pending = [...classSyncState.failed];
                classSyncState.failed = [];
                appendLog(`快速同步完成，开始自然播放兜底：${classSyncState.pending.length} 门`);
            } else {
                classSyncState.active = false;
                classSyncState.current = null;
                appendLog(`班级同步结束：成功 ${classSyncState.results.length}，失败 ${classSyncState.failed.length}`);
                saveClassSyncState();
                return;
            }
        }
        classSyncState.current = classSyncState.pending.shift();
        saveClassSyncState();
        appendLog(`班级同步 ${classSyncState.phase}：${classSyncState.current.name}`);
        window.location.href = classWatchUrl(classSyncState.current);
    }

    async function isCurrentClassTaskPassed() {
        const task = classSyncState.current;
        if (!task) return false;
        const lessons = await fetchClassLessons(task.trainClassId, task.categoryId);
        return lessons.some(item => String(item.tcLessonId) === String(task.tcLessonId) && Number(item.passed) === 1);
    }

    async function completeClassTask(success, reason) {
        const task = classSyncState.current;
        if (!task) return;
        if (success) classSyncState.results.push({ ...task, reason });
        else classSyncState.failed.push({ ...task, reason });
        appendLog(`${success ? '✅' : '⚠'} 班级同步${success ? '成功' : '待重试'}：${task.name}（${reason}）`);
        classSyncState.current = null;
        classSyncState.awaitingExitVerification = false;
        saveClassSyncState();
        setTimeout(processNextClassTask, 800);
    }

    function findExitCourseButton() {
        return Array.from(document.querySelectorAll('button')).find(button =>
            button.offsetParent !== null && button.textContent.trim() === '退出课程'
        );
    }

    async function clickExitAndConfirm() {
        const exit = findExitCourseButton();
        if (!exit) throw new Error('未找到“退出课程”按钮');
        exit.click();
        for (let i = 0; i < 20; i++) {
            const box = document.querySelector('.el-message-box');
            if (box && box.offsetParent !== null && /确定退出当前课程吗？/.test(box.textContent)) {
                const confirmBtn = Array.from(box.querySelectorAll('button')).find(button => button.textContent.trim() === '确定');
                if (!confirmBtn) throw new Error('退出确认框中未找到“确定”按钮');
                confirmBtn.click();
                return;
            }
            await sleep(250);
        }
        throw new Error('未出现退出课程确认对话框');
    }

    async function setupClassSyncVideo(video) {
        forceMute(video);
        const task = classSyncState.current;
        if (!task) return;
        const started = await forcePlayVideo(video);
        if (!started) return completeClassTask(false, '视频未能开始播放');
        const startTime = Number(video.currentTime) || 0;
        await sleep(1200);
        if (video.paused || (Number(video.currentTime) || 0) <= startTime + 0.1) {
            return completeClassTask(false, '未确认视频实际开始播放');
        }
        if (classSyncState.phase === 'quick') {
            try {
                classSyncState.awaitingExitVerification = true;
                saveClassSyncState();
                await clickExitAndConfirm();
                await sleep(1200);
                await completeClassTask(await isCurrentClassTaskPassed(), '快速播放并退出后核验');
            } catch (e) {
                await completeClassTask(false, `快速同步异常：${e.message || e}`);
            }
            return;
        }
        video.addEventListener('ended', async () => {
            await sleep(1200);
            try { await completeClassTask(await isCurrentClassTaskPassed(), '自然播放结束后核验'); }
            catch (e) { await completeClassTask(false, `兜底核验异常：${e.message || e}`); }
        }, { once: true });
    }

    async function resumeClassSyncAfterExit() {
        if (!classSyncState.active || !classSyncState.current || !classSyncState.awaitingExitVerification) return;
        await sleep(1200);
        try {
            await completeClassTask(await isCurrentClassTaskPassed(), '退出课程后的页面跳转核验');
        } catch (e) {
            await completeClassTask(false, `退出后核验异常：${e.message || e}`);
        }
    }


    /* ================================================================
     * watch 页面视频检测
     * ================================================================ */

    async function handleWatchPage() {

        const isClassSyncWatch =
            classSyncState.active &&
            classSyncState.current &&
            new URL(window.location.href).searchParams.get('lessonOrigin') === 'trainclass';

        if (
            !isClassSyncWatch &&
            !loadState('gxela_isLearning', false)
        ) {

            appendLog(
                'watch 页面：当前没有处于自动学习状态'
            );

            return;

        }


        appendLog(
            '进入 watch 页面，开始寻找 video'
        );


        let attempts = 0;


        const detect = async () => {

            if (
                !loadState(
                    'gxela_isLearning',
                    false
                )
            ) {

                return;

            }


            const video =
                findBestVideo();


            if (video) {

                appendLog(
                    `🎬 找到 video：paused=${video.paused} currentTime=${Number(video.currentTime || 0).toFixed(2)}`
                );


                if (isClassSyncWatch) await setupClassSyncVideo(video);
                else await setupVideo(video);


                return;

            }


            attempts++;


            if (
                attempts >=
                VIDEO_DETECT_MAX_RETRIES
            ) {

                appendLog(
                    `❌ ${VIDEO_DETECT_MAX_RETRIES} 次检测仍未找到 video`
                );


                setStatus(
                    '[状态] 未找到视频'
                );


                return;

            }


            appendLog(
                `未找到 video，${VIDEO_DETECT_RETRY_MS / 1000} 秒后重试 ${attempts}/${VIDEO_DETECT_MAX_RETRIES}`
            );


            setTimeout(
                detect,
                VIDEO_DETECT_RETRY_MS
            );

        };


        setTimeout(
            detect,
            800
        );


        /*
         * 网站可能动态创建 video。
         * MutationObserver 可以提前发现。
         */

        const observer =
            new MutationObserver(
                () => {

                    if (
                        currentVideo
                    ) {
                        return;
                    }


                    const video =
                        findBestVideo();


                    if (video) {

                        observer.disconnect();

                        if (isClassSyncWatch) setupClassSyncVideo(video);
                        else setupVideo(video);

                    }

                }
            );


        observer.observe(
            document.documentElement,
            {
                childList: true,
                subtree: true
            }
        );

    }


    /* ================================================================
     * 导出日志
     * ================================================================ */

    function exportLogs() {

        try {

            const blob =
                new Blob(
                    [
                        logs.join('\n')
                    ],
                    {
                        type:
                            'text/plain;charset=utf-8'
                    }
                );


            const url =
                URL.createObjectURL(
                    blob
                );


            const a =
                document.createElement(
                    'a'
                );


            a.href = url;


            a.download =
                `gxela_logs_${new Date()
                    .toISOString()
                    .replace(
                        /[:.]/g,
                        '-'
                    )}.txt`;


            document.body.appendChild(
                a
            );


            a.click();


            a.remove();


            URL.revokeObjectURL(
                url
            );


            appendLog(
                '日志已导出'
            );

        } catch (e) {

            appendLog(
                '日志导出失败：' +
                e
            );

        }

    }


    /* ================================================================
     * 刷新课程列表
     * ================================================================ */

    async function refreshCourseList() {

        try {

            const queue =
                await fetchPlayQueue(
                    currentYear
                );


            renderCourseList();


            appendLog(
                `课程列表刷新完成，共 ${queue.length} 门`
            );

        } catch (e) {

            appendLog(
                '课程列表刷新失败：' +
                e
            );

        }

    }


    /* ================================================================
     * 初始化
     * ================================================================ */

    (async function init() {

        /*
         * 等 DOM
         */

        if (
            document.readyState ===
            'loading'
        ) {

            await new Promise(
                resolve => {

                    document.addEventListener(
                        'DOMContentLoaded',
                        resolve,
                        {
                            once: true
                        }
                    );

                }
            );

        }


        createPanel();


        /*
         * clientid
         */

        injectClientIdInterceptorToPage();


        const scanned =
            scanPageForClientId();


        if (scanned) {

            saveClientId(
                scanned,
                'scan'
            );

        } else {

            appendLog(
                '未检测到页面静态 clientid，继续监听网络请求'
            );

        }


        /*
         * 恢复目标设置
         */

        const targetInputC =
            document.getElementById(
                'gxela-targetCompulsory'
            );

        const targetInputE =
            document.getElementById(
                'gxela-targetElective'
            );

        const yearInput =
            document.getElementById(
                'gxela-year'
            );


        if (targetInputC) {

            targetInputC.value =
                targetHours.compulsory ||
                0;

        }


        if (targetInputE) {

            targetInputE.value =
                targetHours.elective ||
                0;

        }


        if (yearInput) {

            yearInput.value =
                currentYear;

        }


        /*
         * 恢复日志
         */

        const logBox =
            document.getElementById(
                'gxela-logbox'
            );


        if (logBox) {

            logBox.textContent =
                logs.join('\n');

            logBox.scrollTop =
                logBox.scrollHeight;

        }


        restorePlaybackDisplay();


        /*
         * 自动刷新学时
         */

        try {

            await refreshLearnedAndPending(
                currentYear
            );

        } catch (e) {

            appendLog(
                '初始化学时刷新失败：' +
                e
            );

        }


        /*
         * 如果是 watch 页面，
         * 接管视频。
         */

        if (
            window.location.pathname ===
                '/watch' &&
            window.location.search.includes(
                'lessonId='
            )
        ) {

            await handleWatchPage();

        }


        /*
         * 首页则刷新课程列表。
         */

        else if (
            window.location.pathname ===
                '/study/index'
        ) {

            try {

                const learning =
                    loadState(
                        'gxela_isLearning',
                        false
                    );


                if (learning) {

                    await refreshCourseList();

                }

            } catch (e) {

                appendLog(
                    '首页课程列表恢复失败：' +
                    e
                );

            }

            await resumeClassSyncAfterExit();

        }

        else {
            await resumeClassSyncAfterExit();
        }


        appendLog(
            'GXELA V3.6 面板初始化完成'
        );

    })();

})();
