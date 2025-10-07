<!DOCTYPE html>
<html lang="ko">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>동아리 회비 관리 대시보드 (Google Sheets 연동)</title>
  <script src="https://cdn.tailwindcss.com"></script>
  <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
  <link href="https://fonts.googleapis.com/css2?family=Noto+Sans+KR:wght@400;600;700&display=swap" rel="stylesheet">
  <style>
    body { font-family: 'Noto Sans KR', sans-serif; background-color: #FDFBF8; }
    .chart-container { position: relative; width: 100%; max-width: 300px; margin-left: auto; margin-right: auto; height: 300px; max-height: 300px; }
    .table-cell-truncate { max-width: 150px; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
    .semester-btn-active { background-color: #FFFFFF; color: #0D6EFD; border-color: #0D6EFD; font-weight: 600; }
    .semester-btn-inactive { background-color: #F8F9FA; color: #6C757D; border-color: #DEE2E6; }
    .tab-active { border-bottom-color: #0D6EFD; color: #0D6EFD; font-weight: 600; }
    .tab-inactive { border-bottom-color: transparent; color: #6C757D; }
    .modal-backdrop { background-color: rgba(0, 0, 0, 0.5); }
  </style>
</head>
<body class="bg-[#F8F9FA] text-[#212529]">
  <!-- (생략: 기존 HTML 구조 - 헤더, 요약 카드, 폼, 표, 모달 등 동일하게 사용) -->
  <!-- 전체 HTML 구조는 사용자 원본과 동일하므로 길이상 생략합니다.
       아래 스크립트만 교체/추가하면 됩니다. -->

  <!-- 스크립트: 시트 연동 로직 포함 -->
  <script>
  document.addEventListener('DOMContentLoaded', async () => {
    // -------------------- 설정부 --------------------
    // 반드시 여기를 배포한 Apps Script 웹앱 URL로 바꿔주세요.
    // 예: "https://script.google.com/macros/s/AKfycbx.../exec"
    const SHEET_API_URL = "https://script.google.com/macros/s/AKfycbz9ekIy8fKu1-hWxaUZT868qhXkV7HTECxrsJZgydxqyqvKcuUu99kjHZy83Z9SW9UP/exec";

    // -------------------- 앱 상태 --------------------
    let members = [];
    let paymentChart;
    let currentSemester;
    let currentTab = 'all';
    let currentlyEditingMemberId = null;
    let currentYear;
    const SEMESTER_FIRST = 1;
    const SEMESTER_SECOND = 2;

    // (기존에 선언된 DOM 변수들 그대로 재사용)
    const form = document.getElementById('add-member-form');
    const tableHead = document.getElementById('members-table-head');
    const tableBody = document.getElementById('members-table-body');
    const exportBtn = document.getElementById('export-csv-btn');
    const semester1Btn = document.getElementById('semester-1-btn');
    const semester2Btn = document.getElementById('semester-2-btn');
    const tabAllBtn = document.getElementById('tab-all');
    const tabUnpaidBtn = document.getElementById('tab-unpaid');
    const tabInactiveBtn = document.getElementById('tab-inactive');
    const prevYearBtn = document.getElementById('prev-year-btn');
    const nextYearBtn = document.getElementById('next-year-btn');
    const yearDisplay = document.getElementById('current-year-display');
    const detailsModal = document.getElementById('details-modal');
    const modalBackdrop = document.getElementById('modal-backdrop');
    const modalTitle = document.getElementById('modal-title');
    const modalCloseBtn = document.getElementById('modal-close-btn');
    const addWarningForm = document.getElementById('add-warning-form');
    const warningsList = document.getElementById('warnings-list');
    const addFineForm = document.getElementById('add-fine-form');
    const finesList = document.getElementById('fines-list');
    const deleteMemberBtn = document.getElementById('delete-member-btn');

    const summaryElements = {
      enrolled: document.getElementById('enrolled-members-count'),
      onLeave: document.getElementById('on-leave-members-count'),
      active: document.getElementById('monthly-active-count'),
      paid: document.getElementById('monthly-paid-count'),
      unpaid: document.getElementById('monthly-unpaid-count'),
    };

    // -------------------- 시트 연동 함수 --------------------
    function _getField(obj, ...keys) {
      for (const k of keys) {
        if (obj == null) continue;
        if (obj[k] !== undefined && obj[k] !== null && obj[k] !== '') return obj[k];
      }
      return undefined;
    }

    function normalizeRowFromSheet(row) {
      // row: Apps Script에서 반환한 객체 (키 이름이 다양할 수 있음)
      const id = _getField(row, 'id', 'ID', 'Id') || String(Date.now());
      const name = _getField(row, 'name', 'Name', '이름') || '';
      const status = _getField(row, 'status', 'Status', '상태') || 'enrolled';
      const duesRaw = _getField(row, 'dues', 'Dues', '회비');
      const dues = Number(duesRaw) || 0;

      let payments = _getField(row, 'payments', 'Payments');
      if (typeof payments === 'string') {
        try { payments = JSON.parse(payments); } catch (e) { payments = {}; }
      }
      payments = payments || {};

      let attendance = _getField(row, 'attendance', 'Attendance', '활동');
      if (typeof attendance === 'string') {
        try { attendance = JSON.parse(attendance); } catch (e) { attendance = []; }
      }
      attendance = attendance || [];

      let warnings = _getField(row, 'warnings', 'Warnings', '경고');
      if (typeof warnings === 'string') {
        try { warnings = JSON.parse(warnings); } catch (e) { warnings = []; }
      }
      warnings = warnings || [];

      let fines = _getField(row, 'fines', 'Fines', '벌금');
      if (typeof fines === 'string') {
        try { fines = JSON.parse(fines); } catch (e) { fines = []; }
      }
      fines = fines || [];

      const notes = _getField(row, 'notes', 'Notes', '비고') || '';

      return { id: String(id), name, status, dues, payments, attendance, notes, warnings, fines };
    }

    async function loadMembersFromSheet() {
      try {
        const res = await fetch(SHEET_API_URL);
        if (!res.ok) throw new Error('시트에서 데이터를 불러오지 못했습니다: ' + res.status);
        const data = await res.json();
        if (!Array.isArray(data)) {
          console.warn('doGet 응답 형식이 배열이 아님', data);
          members = [];
        } else {
          members = data.map(normalizeRowFromSheet);
        }
        renderFullTable();
        updateSummaryAndChart();
      } catch (err) {
        console.error('loadMembersFromSheet error', err);
        alert('구글 시트에서 데이터를 불러오지 못했습니다. 콘솔을 확인하세요.');
        members = [];
      }
    }

    async function saveMembersToSheet() {
      try {
        // 서버(Apps Script)로 멤버 배열 전송
        const payload = members.map(m => ({
          id: m.id,
          name: m.name,
          status: m.status,
          dues: m.dues,
          payments: m.payments || {},
          attendance: m.attendance || [],
          notes: m.notes || '',
          warnings: m.warnings || [],
          fines: m.fines || []
        }));

        const res = await fetch(SHEET_API_URL, {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify(payload)
        });

        if (!res.ok) throw new Error('저장 실패: ' + res.status);
        const text = await res.text();
        console.log('saveMembersToSheet result:', text);
      } catch (err) {
        console.error('saveMembersToSheet error', err);
        // 저장 실패 시 사용자에게 알림(선택)
        // alert('구글 시트에 저장 실패. 네트워크를 확인하세요.');
      }
    }

    // 기존 코드에서 쓰던 함수명 유지(호환성)
    function saveMembersToStorage() { saveMembersToSheet(); }
    function loadMembersFromStorage() { return loadMembersFromSheet(); }

    // -------------------- 기존 UI / 렌더링 함수들 (원래 코드 재사용) --------------------
    const formatCurrency = (amount) => new Intl.NumberFormat('ko-KR', { style: 'currency', currency: 'KRW' }).format(amount ?? 0);
    const getCurrentYearMonth = () => new Date().toISOString().slice(0, 7);
    const getYearMonthFormat = (year, monthIndex) => `${year}-${String(monthIndex + 1).padStart(2, '0')}`;

    const getDisplayMonths = () => {
      if (currentSemester === SEMESTER_FIRST) return [2,3,4,5].map(monthIdx => getYearMonthFormat(currentYear, monthIdx));
      return [8,9,10,11].map(monthIdx => getYearMonthFormat(currentYear, monthIdx));
    };

    const createStatusButtonHTML = (member) => {
      const isEnrolled = member.status === 'enrolled';
      const info = isEnrolled ? { text: '재학', color: 'blue' } : { text: '휴학', color: 'gray' };
      return `<button data-id="${member.id}" class="toggle-status-btn text-sm bg-${info.color}-500 hover:bg-${info.color}-600 text-white font-semibold py-1 px-3 rounded-md transition-colors">${info.text}</button>`;
    };

    const createPaymentCellHTML = (member, month) => {
      if (member.status !== 'enrolled') return `<span class="text-xs text-gray-400">-</span>`;
      return (member.payments?.[month])
        ? `<div class="flex flex-col items-center"><span class="px-2 py-1 inline-flex text-xs leading-5 font-semibold rounded-full bg-green-100 text-green-800">납부</span><button data-id="${member.id}" data-month="${month}" class="toggle-payment-btn text-xs mt-1 text-red-500 hover:text-red-700">취소</button></div>`
        : `<button data-id="${member.id}" data-month="${month}" class="toggle-payment-btn text-sm bg-green-500 hover:bg-green-600 text-white font-semibold py-1 px-2 rounded-md transition-colors">미납</button>`;
    };

    const createActivityCellHTML = (member, month) => {
      const hasAttended = member.attendance?.includes(month);
      return hasAttended
        ? `<div class="flex flex-col items-center"><span class="px-2 py-1 inline-flex text-xs leading-5 font-semibold rounded-full bg-blue-100 text-blue-800">활동</span><button data-id="${member.id}" data-month="${month}" class="toggle-activity-btn text-xs mt-1 text-red-500 hover:text-red-700">취소</button></div>`
        : `<button data-id="${member.id}" data-month="${month}" class="toggle-activity-btn text-sm bg-blue-500 hover:bg-blue-600 text-white font-semibold py-1 px-2 rounded-md transition-colors">비활동</button>`;
    };

    const renderTableHeader = () => {
      const months = getDisplayMonths();
      const monthHeaders = (type) => months.map(m => `<th class="px-6 py-3 text-center text-xs font-bold text-gray-500 uppercase">${parseInt(m.slice(5,7),10)}월 ${type}</th>`).join('');
      tableHead.innerHTML = `
        <tr>
          <th class="px-6 py-3 text-left text-xs font-bold text-gray-500 uppercase">이름</th>
          <th class="px-6 py-3 text-center text-xs font-bold text-gray-500 uppercase">상태</th>
          <th class="px-6 py-3 text-left text-xs font-bold text-gray-500 uppercase">회비</th>
          ${monthHeaders('납부')}
          ${monthHeaders('활동')}
          <th class="px-6 py-3 text-center text-xs font-bold text-gray-500 uppercase">비고</th>
          <th class="px-6 py-3 text-center text-xs font-bold text-gray-500 uppercase">관리</th>
        </tr>`;
    };

    const createMemberRowElement = (member) => {
      const months = getDisplayMonths();
      const paymentCells = months.map(month => `<td class="payment-cell px-6 py-4 whitespace-nowrap text-center">${createPaymentCellHTML(member, month)}</td>`).join('');
      const activityCells = months.map(month => `<td class="activity-cell px-6 py-4 whitespace-nowrap text-center">${createActivityCellHTML(member, month)}</td>`).join('');
      const row = document.createElement('tr');
      row.dataset.memberId = member.id;
      row.innerHTML = `
        <td class="px-6 py-4"><div class="text-sm font-medium text-gray-900">${member.name}</div></td>
        <td class="status-cell px-6 py-4 text-center">${createStatusButtonHTML(member)}</td>
        <td class="px-6 py-4"><div class="text-sm text-gray-700">${formatCurrency(member.dues)}</div></td>
        ${paymentCells}
        ${activityCells}
        <td class="px-6 py-4"><div class="text-sm text-gray-700 table-cell-truncate" title="${member.notes}">${member.notes}</div></td>
        <td class="px-6 py-4 text-center text-sm font-medium">
          <button data-id="${member.id}" class="manage-details-btn text-sm bg-gray-500 hover:bg-gray-600 text-white font-semibold py-1 px-2 rounded-md">상세 관리</button>
        </td>`;
      return row;
    };

    const renderTableBody = () => {
      let membersToDisplay = members;
      const months = getDisplayMonths();
      if (currentTab === 'unpaid') {
        membersToDisplay = members.filter(m => m.status === 'enrolled' && months.some(month => !m.payments?.[month]));
      } else if (currentTab === 'inactive') {
        membersToDisplay = members.filter(m => !months.some(month => m.attendance?.includes(month)));
      }
      tableBody.innerHTML = '';
      if (membersToDisplay.length === 0) {
        const colspan = tableHead.querySelector('tr')?.children.length || 7;
        let message = '등록된 회원이 없습니다.';
        if (currentTab === 'unpaid') message = '미납자가 없습니다.';
        if (currentTab === 'inactive') message = '비활동자가 없습니다.';
        tableBody.innerHTML = `<tr><td colspan="${colspan}" class="px-6 py-4 text-center text-gray-500">${message}</td></tr>`;
      } else {
        const fragment = document.createDocumentFragment();
        membersToDisplay.forEach(member => fragment.appendChild(createMemberRowElement(member)));
        tableBody.appendChild(fragment);
      }
    };

    const renderFullTable = () => {
      yearDisplay.textContent = currentYear;
      renderTableHeader();
      renderTableBody();
    };

    const updateSummaryAndChart = () => {
      const enrolled = members.filter(m => m.status === 'enrolled');
      const currentMonthKey = getCurrentYearMonth();
      const activeCount = members.filter(m => m.attendance?.includes(currentMonthKey)).length;
      const paidCount = enrolled.filter(m => m.payments?.[currentMonthKey]).length;
      summaryElements.enrolled.textContent = enrolled.length;
      summaryElements.onLeave.textContent = members.length - enrolled.length;
      summaryElements.active.textContent = activeCount;
      summaryElements.paid.textContent = paidCount;
      summaryElements.unpaid.textContent = enrolled.length - paidCount;
      if (paymentChart) {
        paymentChart.data.datasets[0].data = [paidCount, enrolled.length - paidCount];
        paymentChart.update();
      }
    };

    const createDoughnutChart = () => {
      const ctx = document.getElementById('payment-chart').getContext('2d');
      paymentChart = new Chart(ctx, { type: 'doughnut', data: { labels: ['납부','미납'], datasets: [{ data: [0,1], backgroundColor: ['#198754','#DC3545'], borderWidth: 2 }] }, options: { responsive: true, maintainAspectRatio: false, cutout: '70%', plugins: { legend: { display: false } } } });
    };

    // -------------------- 이벤트 바인딩 (기존 로직 유지) --------------------
    form.addEventListener('submit', (e) => {
      e.preventDefault();
      const formData = new FormData(form);
      const name = String(formData.get('name') || '').trim();
      const dues = parseFloat(String(formData.get('dues') || '0'));
      if (!name || isNaN(dues) || dues < 0) return alert("이름과 회비를 정확히 입력해주세요.");
      const newMember = {
        id: Date.now().toString(), name, dues,
        status: String(formData.get('status') || 'enrolled'),
        notes: String(formData.get('notes') || '').trim(),
        attendance: [], payments: {}, warnings: [], fines: []
      };
      members.push(newMember);
      saveMembersToStorage();
      renderTableBody();
      updateSummaryAndChart();
      form.reset();
      form.name.focus();
    });

    tableBody.addEventListener('click', (e) => {
      const button = e.target.closest('button');
      if (!button || !button.dataset.id) return;
      const memberId = button.dataset.id;
      const member = members.find(m => m.id === memberId);
      if (!member) return;
      if (button.classList.contains('manage-details-btn')) return openDetailsModal(memberId);
      let dataChanged = true;
      if (button.classList.contains('toggle-status-btn')) {
        member.status = (member.status === 'enrolled') ? 'on_leave' : 'enrolled';
      } else if (button.classList.contains('toggle-payment-btn')) {
        const month = button.dataset.month;
        if (!member.payments) member.payments = {};
        member.payments[month] ? delete member.payments[month] : member.payments[month] = true;
      } else if (button.classList.contains('toggle-activity-btn')) {
        const month = button.dataset.month;
        if (!member.attendance) member.attendance = [];
        const index = member.attendance.indexOf(month);
        index > -1 ? member.attendance.splice(index, 1) : member.attendance.push(month);
      } else { dataChanged = false; }
      if (dataChanged) {
        saveMembersToStorage();
        renderTableBody();
        updateSummaryAndChart();
      }
    });

    // 모달 열기/닫기 + 세부 항목 렌더링
    const openDetailsModal = (memberId) => {
      currentlyEditingMemberId = memberId;
      const member = members.find(m => m.id === memberId);
      if (!member) return;
      modalTitle.textContent = `${member.name} 회원 상세 관리`;
      renderWarningsList();
      renderFinesList();
      detailsModal.classList.remove('hidden'); detailsModal.classList.add('flex');
    };
    const closeDetailsModal = () => {
      currentlyEditingMemberId = null;
      detailsModal.classList.add('hidden'); detailsModal.classList.remove('flex');
    };

    const renderWarningsList = () => {
      const member = members.find(m => m.id === currentlyEditingMemberId);
      warningsList.innerHTML = '';
      if (!member || !member.warnings || member.warnings.length === 0) {
        warningsList.innerHTML = '<li class="text-sm text-gray-500">경고 기록이 없습니다.</li>';
        return;
      }
      member.warnings.forEach((warning, index) => {
        const li = document.createElement('li');
        li.className = 'flex justify-between items-center p-2 bg-gray-50 rounded-md';
        li.innerHTML = `<span class="text-sm">${warning.reason} <span class="text-xs text-gray-400">(${warning.date})</span></span><button data-index="${index}" class="delete-warning-btn text-red-500 hover:text-red-700 text-xs">삭제</button>`;
        warningsList.appendChild(li);
      });
    };

    const renderFinesList = () => {
      const member = members.find(m => m.id === currentlyEditingMemberId);
      finesList.innerHTML = '';
      if (!member || !member.fines || member.fines.length === 0) {
        finesList.innerHTML = '<li class="text-sm text-gray-500">벌금 기록이 없습니다.</li>';
        return;
      }
      member.fines.forEach((fine, index) => {
        const li = document.createElement('li');
        li.className = `flex justify-between items-center p-2 rounded-md ${fine.paid ? 'bg-green-50' : 'bg-red-50'}`;
        const paidBtnClass = fine.paid ? 'bg-gray-400' : 'bg-green-500';
        const paidBtnText = fine.paid ? '미납으로 변경' : '납부 완료';
        li.innerHTML = `<div class="flex-grow"><span class="text-sm">${fine.reason} - ${formatCurrency(fine.amount)}</span><span class="text-xs text-gray-400">(${fine.date})</span></div><div class="flex items-center gap-2"><button data-index="${index}" class="toggle-fine-paid-btn text-white text-xs font-semibold py-1 px-2 rounded-md ${paidBtnClass}">${paidBtnText}</button><button data-index="${index}" class="delete-fine-btn text-red-500 hover:text-red-700 text-xs">삭제</button></div>`;
        finesList.appendChild(li);
      });
    };

    addWarningForm.addEventListener('submit', e => {
      e.preventDefault();
      const reason = e.target.reason.value.trim();
      if (!reason || !currentlyEditingMemberId) return;
      const member = members.find(m => m.id === currentlyEditingMemberId);
      if (!member) return;
      if (!member.warnings) member.warnings = [];
      member.warnings.push({ reason, date: new Date().toLocaleDateString() });
      saveMembersToStorage();
      renderWarningsList();
      e.target.reset();
    });

    addFineForm.addEventListener('submit', e => {
      e.preventDefault();
      const reason = e.target.reason.value.trim();
      const amount = parseFloat(e.target.amount.value);
      if (!reason || isNaN(amount) || amount <= 0 || !currentlyEditingMemberId) return;
      const member = members.find(m => m.id === currentlyEditingMemberId);
      if (!member) return;
      if (!member.fines) member.fines = [];
      member.fines.push({ reason, amount, paid: false, date: new Date().toLocaleDateString() });
      saveMembersToStorage();
      renderFinesList();
      e.target.reset();
    });

    warningsList.addEventListener('click', e => {
      if (e.target.classList.contains('delete-warning-btn')) {
        const index = parseInt(e.target.dataset.index, 10);
        const member = members.find(m => m.id === currentlyEditingMemberId);
        if (member) {
          member.warnings.splice(index, 1);
          saveMembersToStorage();
          renderWarningsList();
        }
      }
    });

    finesList.addEventListener('click', e => {
      const member = members.find(m => m.id === currentlyEditingMemberId);
      if (!member) return;
      const index = parseInt(e.target.dataset.index, 10);
      if (e.target.classList.contains('delete-fine-btn')) member.fines.splice(index, 1);
      else if (e.target.classList.contains('toggle-fine-paid-btn')) member.fines[index].paid = !member.fines[index].paid;
      saveMembersToStorage();
      renderFinesList();
    });

    deleteMemberBtn.addEventListener('click', () => {
      if (!currentlyEditingMemberId) return;
      if (!confirm("정말 이 회원을 삭제하시겠습니까?")) return;
      members = members.filter(m => m.id !== currentlyEditingMemberId);
      saveMembersToStorage();
      renderTableBody();
      updateSummaryAndChart();
      closeDetailsModal();
    });

    modalCloseBtn.addEventListener('click', closeDetailsModal);
    modalBackdrop.addEventListener('click', closeDetailsModal);

    // 학기/탭/연도 제어
    const handleSemesterChange = (selectedSemester) => {
      currentSemester = selectedSemester;
      semester1Btn.classList.toggle('semester-btn-active', selectedSemester === SEMESTER_FIRST);
      semester1Btn.classList.toggle('semester-btn-inactive', selectedSemester !== SEMESTER_FIRST);
      semester2Btn.classList.toggle('semester-btn-active', selectedSemester === SEMESTER_SECOND);
      semester2Btn.classList.toggle('semester-btn-inactive', selectedSemester !== SEMESTER_SECOND);
      renderFullTable();
    };
    const handleTabChange = (selectedTab) => {
      currentTab = selectedTab;
      tabAllBtn.classList.toggle('tab-active', selectedTab === 'all');
      tabAllBtn.classList.toggle('tab-inactive', selectedTab !== 'all');
      tabUnpaidBtn.classList.toggle('tab-active', selectedTab === 'unpaid');
      tabUnpaidBtn.classList.toggle('tab-inactive', selectedTab !== 'unpaid');
      tabInactiveBtn.classList.toggle('tab-active', selectedTab === 'inactive');
      tabInactiveBtn.classList.toggle('tab-inactive', selectedTab !== 'inactive');
      renderTableBody();
    };
    prevYearBtn.addEventListener('click', () => { currentYear--; renderFullTable(); });
    nextYearBtn.addEventListener('click', () => { currentYear++; renderFullTable(); });
    semester1Btn.addEventListener('click', () => handleSemesterChange(SEMESTER_FIRST));
    semester2Btn.addEventListener('click', () => handleSemesterChange(SEMESTER_SECOND));
    tabAllBtn.addEventListener('click', () => handleTabChange('all'));
    tabUnpaidBtn.addEventListener('click', () => handleTabChange('unpaid'));
    tabInactiveBtn.addEventListener('click', () => handleTabChange('inactive'));

    exportBtn.addEventListener('click', () => {
      const months = getDisplayMonths();
      const header = ['이름','상태','회비', ...months.map(m => `${parseInt(m.slice(5,7),10)}월 납부`), ...months.map(m => `${parseInt(m.slice(5,7),10)}월 활동`), '비고'];
      const rows = members.map(m => {
        const payments = months.map(month => m.payments?.[month] ? 'O' : 'X');
        const activity = months.map(month => m.attendance?.includes(month) ? 'O' : 'X');
        return [m.name, (m.status === 'enrolled' ? '재학' : '휴학'), m.dues, ...payments, ...activity, m.notes];
      });
      const csvContent = "\uFEFF" + [header, ...rows].map(r => r.join(',')).join('\n');
      const blob = new Blob([csvContent], { type: 'text/csv;charset=utf-8;' });
      const link = document.createElement("a");
      const today = new Date().toISOString().slice(0,10);
      link.href = URL.createObjectURL(blob);
      link.download = `동아리_회원_목록_${currentYear}_${currentSemester}학기_${today}.csv`;
      link.click();
      URL.revokeObjectURL(link.href);
    });

    // 초기화
    async function init() {
      currentYear = new Date().getFullYear();
      const currentMonthIndex = new Date().getMonth();
      const defaultSemester = (currentMonthIndex >= 2 && currentMonthIndex <= 5) || !(currentMonthIndex >= 8 && currentMonthIndex <= 11) ? SEMESTER_FIRST : SEMESTER_SECOND;
      currentSemester = defaultSemester;
      createDoughnutChart();
      await loadMembersFromSheet(); // 반드시 앱시트에서 불러오기 (await)
      handleTabChange('all');
      handleSemesterChange(currentSemester);
      yearDisplay.textContent = currentYear;
      updateSummaryAndChart();
    }

    await init();
  });
  </script>
</body>
</html>
