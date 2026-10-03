<title>3지역 종합 대시보드</title> <script src="https://cdn.jsdelivr.net/npm/chart.js"></script> <style> :root { --bg: #0d1117; --card: #161b22; --border: #30363d; --text: #c9d1d9; --neon: #58a6ff; --highlight: #2ea043; --warning: #d29922; --accent: #a371f7; --gray: #8b949e; --danger: #f85149; } body { background-color: var(--bg); color: var(--text); font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; margin: 0; padding: 20px; } h1, h2, h3 { color: #ffffff; text-align: center; }
    .summary-container {
        display: grid;
        grid-template-columns: repeat(4, 1fr);
        gap: 15px;
        margin-bottom: 30px;
    }
    .kpi-card {
        background: var(--card); border: 1px solid var(--border);
        padding: 20px; border-radius: 8px; text-align: center;
    }
    .kpi-value {
        font-size: 32px; font-weight: bold; color: var(--neon); margin-top: 10px;
    }
    #total-score-val { color: var(--highlight); }

    .section-box {
        background: var(--card); border: 1px solid var(--border);
        border-radius: 8px; padding: 20px; margin-bottom: 30px;
    }
    .section-box h2 { border-bottom: 1px solid var(--border); padding-bottom: 10px; margin-top: 0; }

    .chart-grid {
        display: grid; grid-template-columns: repeat(auto-fit, minmax(400px, 1fr));
        gap: 20px; margin-top: 20px;
    }
    .chart-box {
        background: var(--bg); border: 1px solid var(--border);
        border-radius: 8px; padding: 15px; height: 330px; position: relative;
    }
    .chart-box h3 { 
        margin-top: 0; font-size: 16px; color: var(--gray); margin-bottom: 15px; 
        text-align: left; display: flex; justify-content: space-between; align-items: center;
    }
    .delta-badge {
        font-size: 13px; padding: 4px 8px; border-radius: 12px; background: rgba(255, 255, 255, 0.05);
    }
    .trend-chart-container { width: 100%; height: 300px; position: relative; }
</style>
<h1 id="dashboard-title">3지역 주간 종합 대시보드 (로딩중...)</h1>

<div class="summary-container">
    <div class="kpi-card"><h3>총 재적</h3><div class="kpi-value">115명</div></div>
    <div class="kpi-card"><h3>출결 제외</h3><div class="kpi-value">2명</div></div>
    <div class="kpi-card"><h3>출결 재적</h3><div class="kpi-value">113명</div></div>
    <div class="kpi-card"><h3>이번주 전방부 총점</h3><div id="total-score-val" class="kpi-value">0점</div></div>
</div>

<div class="section-box">
    <h2>📈 주차별 전방부 종합점수 추이</h2>
    <div class="trend-chart-container">
        <canvas id="trendChart"></canvas>
    </div>
</div>

<div class="section-box">
    <h2>🙏 이번주 예배 및 모임 현황 (지난주 대비 증감)</h2>
    <div class="chart-grid" id="worship-charts-container">
        </div>
</div>

<div class="section-box">
    <h2>📊 평가 항목별 구역 세부 지표 (지난주 대비 증감)</h2>
    <div class="chart-grid" id="bar-charts-container">
        </div>
</div>

<script>
    const sheetUrl = 'https://docs.google.com/spreadsheets/d/e/2PACX-1vSjhl42zs2Zd_xygVqvJ1Dwcls5YqU0YZU31sgqE2XeO_5LWodLkjjKkO-WmzLKeuja4Sriodynh59c/pub?output=csv';

    // 1. 예배 및 모임 현황 설정 (점수 없음)
    const worshipCategories = [
        { id: 'w1', name: '예배 참석', colIndex: 2, color: 'rgba(46, 160, 67, 0.8)' }, // 초록
        { id: 'w2', name: '줌 예배', colIndex: 3, color: 'rgba(210, 153, 34, 0.8)' }, // 노랑
        { id: 'w3', name: '결석', colIndex: 4, color: 'rgba(248, 81, 73, 0.8)' },   // 빨강
        { id: 'w4', name: '심방률', colIndex: 11, color: 'rgba(163, 113, 247, 0.8)' },// 보라
        { id: 'w5', name: '구역예배', colIndex: 12, color: 'rgba(88, 166, 255, 0.8)' } // 파랑
    ];

    // 2. 전방부 지표 설정 (점수 있음)
    const scoreCategories = [
        { id: 'cat1', name: '센터등록', colIndex: 5, points: 50 },
        { id: 'cat2', name: '실교사', colIndex: 6, points: 10 },
        { id: 'cat3', name: '활동자', colIndex: 7, points: 1 },
        { id: 'cat4', name: '섬김이', colIndex: 8, points: 5 },
        { id: 'cat5', name: '신임사명자 양성', colIndex: 9, points: 20 },
        { id: 'cat6', name: '성공사례발표', colIndex: 10, points: 10 }
    ];

    Chart.defaults.color = '#8b949e';
    Chart.defaults.borderColor = '#30363d';

    async function fetchSheetData() {
        try {
            const response = await fetch(sheetUrl);
            const data = await response.text();
            
            const rows = data.split('\n').map(row => row.trim()).filter(row => row.length > 0);
            if (rows.length < 2) return;

            const weeklyData = {};
            const weekLabels = [];
            const weekScores = [];

            rows.slice(1).forEach(row => {
                const cols = row.split(',');
                const week = cols[0] ? cols[0].trim() : '';
                if(!week) return;

                if (!weeklyData[week]) {
                    weeklyData[week] = [];
                    weekLabels.push(week);
                }
                weeklyData[week].push(cols);
            });

            // 주차별 총점 계산
            weekLabels.forEach(week => {
                let wScore = 0;
                weeklyData[week].forEach(cols => {
                    scoreCategories.forEach(cat => {
                        const count = parseInt(cols[cat.colIndex]) || 0;
                        wScore += count * cat.points;
                    });
                });
                weekScores.push(wScore);
            });

            drawTrendChart(weekLabels, weekScores);

            // 최신 주차 및 지난주 데이터 세팅
            const latestWeekIndex = weekLabels.length - 1;
            const latestWeek = weekLabels[latestWeekIndex];
            const prevWeek = latestWeekIndex > 0 ? weekLabels[latestWeekIndex - 1] : null;

            const latestRows = weeklyData[latestWeek];
            const prevRows = prevWeek ? weeklyData[prevWeek] : [];
            
            document.getElementById('dashboard-title').innerText = `3지역 종합 대시보드 (${latestWeek})`;

            // 지난주 데이터를 구역 이름으로 맵핑
            const prevDataMap = {};
            prevRows.forEach(cols => {
                const gName = cols[1] ? cols[1].trim() : '-';
                prevDataMap[gName] = cols;
            });

            let grandTotalScore = 0;
            const groupNames = []; 

            // 데이터 분류 배열 생성
            const wCounts = [[], [], [], [], []];      // 이번주 예배 데이터
            const prevWCounts = [[], [], [], [], []];  // 지난주 예배 데이터
            const wTotals = [0, 0, 0, 0, 0];
            const prevWTotals = [0, 0, 0, 0, 0];

            const sCounts = [[], [], [], [], [], []];  // 이번주 지표 데이터
            const prevSCounts = [[], [], [], [], [], []]; // 지난주 지표 데이터
            const sTotals = [0, 0, 0, 0, 0, 0];
            const prevSTotals = [0, 0, 0, 0, 0, 0];

            // 행 단위 데이터 파싱
            latestRows.forEach(cols => {
                const groupName = cols[1] ? cols[1].trim() : '-';
                groupNames.push(groupName);
                const prevCols = prevDataMap[groupName] || null;

                // 예배 데이터 추출
                worshipCategories.forEach((cat, idx) => {
                    const count = parseInt(cols[cat.colIndex]) || 0;
                    const prevCount = prevCols ? (parseInt(prevCols[cat.colIndex]) || 0) : 0;
                    wCounts[idx].push(count);
                    prevWCounts[idx].push(prevCount);
                    wTotals[idx] += count;
                    prevWTotals[idx] += prevCount;
                });

                // 지표 데이터 추출 및 총점 계산
                let groupTotal = 0;
                scoreCategories.forEach((cat, idx) => {
                    const count = parseInt(cols[cat.colIndex]) || 0;
                    const prevCount = prevCols ? (parseInt(prevCols[cat.colIndex]) || 0) : 0;
                    groupTotal += count * cat.points;
                    
                    sCounts[idx].push(count); 
                    prevSCounts[idx].push(prevCount);
                    sTotals[idx] += count;
                    prevSTotals[idx] += prevCount;
                });
                grandTotalScore += groupTotal;
            });

            document.getElementById('total-score-val').innerText = grandTotalScore + '점';

            // 공통 차트 렌더링 함수
            function renderCharts(containerId, categoryMeta, thisData, prevData, totals, prevTotals, defaultColor) {
                const container = document.getElementById(containerId);
                container.innerHTML = ''; 

                categoryMeta.forEach((cat, idx) => {
                    const diff = totals[idx] - prevTotals[idx];
                    let diffHtml = `<span style="color:var(--gray);">-</span>`;
                    if (prevWeek) {
                        if (diff > 0) diffHtml = `<span style="color:var(--highlight);">▲ ${diff}</span>`;
                        else if (diff < 0) diffHtml = `<span style="color:var(--danger);">▼ ${Math.abs(diff)}</span>`;
                    }

                    const titleSuffix = cat.points ? ` (${cat.points}점)` : '';
                    
                    const chartDiv = document.createElement('div');
                    chartDiv.className = 'chart-box';
                    chartDiv.innerHTML = `
                        <h3>
                            <span>${cat.name}${titleSuffix}</span>
                            <span class="delta-badge">총 ${totals[idx]} ${prevWeek ? `(${diffHtml})` : ''}</span>
                        </h3>
                        <canvas id="chart-${cat.id}"></canvas>
                    `;
                    container.appendChild(chartDiv);

                    const ctx = document.getElementById(`chart-${cat.id}`).getContext('2d');
                    new Chart(ctx, {
                        type: 'bar',
                        data: {
                            labels: groupNames, 
                            datasets: [
                                {
                                    label: '지난주',
                                    data: prevData[idx],
                                    backgroundColor: 'rgba(139, 148, 158, 0.3)', // 회색 (지난주)
                                    borderRadius: 4
                                },
                                {
                                    label: '이번주',
                                    data: thisData[idx],
                                    backgroundColor: cat.color || defaultColor, // 각 항목별 색상 적용
                                    borderRadius: 4
                                }
                            ]
                        },
                        options: {
                            responsive: true,
                            maintainAspectRatio: false,
                            plugins: { legend: { display: true, position: 'bottom', labels: { boxWidth: 12 } } },
                            scales: { y: { beginAtZero: true, ticks: { stepSize: 1 } } }
                        }
                    });
                });
            }

            // 예배 및 모임 현황 차트 렌더링 (worshipCategories에 지정된 색상 사용)
            renderCharts('worship-charts-container', worshipCategories, wCounts, prevWCounts, wTotals, prevWTotals, null);
            
            // 전방부 지표 차트 렌더링 (기본 파란색 사용)
            renderCharts('bar-charts-container', scoreCategories, sCounts, prevSCounts, sTotals, prevSTotals, 'rgba(88, 166, 255, 0.8)');

        } catch (error) {
            console.error('데이터 오류:', error);
        }
    }

    function drawTrendChart(labels, data) {
        const ctx = document.getElementById('trendChart').getContext('2d');
        new Chart(ctx, {
            type: 'line',
            data: {
                labels: labels,
                datasets: [{
                    label: '전방부 총점',
                    data: data,
                    borderColor: '#2ea043',
                    backgroundColor: 'rgba(46, 160, 67, 0.1)',
                    borderWidth: 2,
                    pointBackgroundColor: '#2ea043',
                    fill: true,
                    tension: 0.3
                }]
            },
            options: {
                responsive: true,
                maintainAspectRatio: false,
                plugins: { legend: { display: false } }
            }
        });
    }

    window.onload = fetchSheetData;
</script>

