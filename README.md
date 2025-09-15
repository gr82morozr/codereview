"""

~~~


<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Holiday Planner - Australia</title>
    <style>
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
        }
        
        body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            min-height: 100vh;
            padding: 20px;
        }
        
        .container {
            max-width: 1200px;
            margin: 0 auto;
            background: white;
            border-radius: 20px;
            box-shadow: 0 20px 40px rgba(0,0,0,0.1);
            overflow: hidden;
        }
        
        .header {
            background: linear-gradient(135deg, #4facfe 0%, #00f2fe 100%);
            color: white;
            padding: 30px;
            text-align: center;
        }
        
        .header h1 {
            font-size: 2.5rem;
            margin-bottom: 10px;
            font-weight: 600;
        }
        
        .header p {
            font-size: 1.1rem;
            opacity: 0.9;
        }
        
        .calendar-container {
            display: flex;
            gap: 20px;
            padding: 30px;
            justify-content: center;
            flex-wrap: wrap;
        }
        
        .month-calendar {
            background: #f8f9ff;
            border-radius: 15px;
            padding: 20px;
            box-shadow: 0 10px 25px rgba(0,0,0,0.05);
            min-width: 350px;
        }
        
        .month-title {
            text-align: center;
            font-size: 1.5rem;
            font-weight: 600;
            color: #333;
            margin-bottom: 20px;
        }
        
        .calendar-grid {
            display: grid;
            grid-template-columns: repeat(7, 1fr);
            gap: 2px;
            background: #e0e7ff;
            border-radius: 10px;
            padding: 5px;
        }
        
        .day-header {
            background: #6366f1;
            color: white;
            text-align: center;
            padding: 12px 8px;
            font-weight: 600;
            font-size: 0.9rem;
        }
        
        .day-header:first-child {
            border-top-left-radius: 5px;
        }
        
        .day-header:last-child {
            border-top-right-radius: 5px;
        }
        
        .day-cell {
            background: white;
            text-align: center;
            padding: 12px 8px;
            cursor: pointer;
            transition: all 0.3s ease;
            border: 2px solid transparent;
            font-weight: 500;
            position: relative;
        }
        
        .day-cell:hover {
            background: #f0f4ff;
            transform: scale(1.05);
        }
        
        .day-cell.other-month {
            color: #ccc;
            background: #f5f5f5;
        }
        
        .day-cell.weekend {
            background: #fef3c7;
            color: #92400e;
        }
        
        .day-cell.public-holiday {
            background: #fecaca;
            color: #dc2626;
            font-weight: 600;
        }
        
        .day-cell.selected-start {
            background: #10b981;
            color: white;
            border-color: #059669;
        }
        
        .day-cell.selected-range {
            background: #d1fae5;
            color: #065f46;
        }
        
        .day-cell.selected-end {
            background: #ef4444;
            color: white;
            border-color: #dc2626;
        }
        
        .info-panel {
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            color: white;
            padding: 30px;
            text-align: center;
        }
        
        .legend {
            display: flex;
            justify-content: center;
            gap: 30px;
            margin-bottom: 20px;
            flex-wrap: wrap;
        }
        
        .legend-item {
            display: flex;
            align-items: center;
            gap: 8px;
        }
        
        .legend-color {
            width: 20px;
            height: 20px;
            border-radius: 4px;
            border: 2px solid rgba(255,255,255,0.3);
        }
        
        .holiday-info {
            background: rgba(255,255,255,0.1);
            border-radius: 10px;
            padding: 20px;
            margin-top: 20px;
            backdrop-filter: blur(10px);
        }
        
        .holiday-info h3 {
            margin-bottom: 10px;
            font-size: 1.3rem;
        }
        
        .holiday-details {
            display: grid;
            grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
            gap: 15px;
            margin-top: 15px;
        }
        
        .detail-item {
            background: rgba(255,255,255,0.1);
            padding: 10px;
            border-radius: 5px;
            text-align: left;
        }
        
        .detail-label {
            font-weight: 600;
            margin-bottom: 5px;
        }
        
        @media (max-width: 768px) {
            .calendar-container {
                flex-direction: column;
            }
            
            .legend {
                flex-direction: column;
                align-items: center;
            }
            
            .header h1 {
                font-size: 2rem;
            }
        }
    </style>
</head>
<body>
    <div class="container">
        <div class="header">
            <h1>🏖️ Holiday Planner</h1>
            <p>Plan your perfect Australian holiday - December 2025 & January 2026</p>
        </div>
        
        <div class="calendar-container">
            <div class="month-calendar">
                <div class="month-title">December 2025</div>
                <div class="calendar-grid" id="december-calendar">
                    <!-- December calendar will be generated here -->
                </div>
            </div>
            
            <div class="month-calendar">
                <div class="month-title">January 2026</div>
                <div class="calendar-grid" id="january-calendar">
                    <!-- January calendar will be generated here -->
                </div>
            </div>
        </div>
        
        <div class="info-panel">
            <div class="legend">
                <div class="legend-item">
                    <div class="legend-color" style="background: #fef3c7;"></div>
                    <span>Weekend</span>
                </div>
                <div class="legend-item">
                    <div class="legend-color" style="background: #8b5cf6; border: 2px solid #7c3aed;"></div>
                    <span>Public Holiday</span>
                </div>
                <div class="legend-item">
                    <div class="legend-color" style="background: white; border: 2px solid #10b981; color: #10b981; font-weight: bold; display: flex; align-items: center; justify-content: center; font-size: 12px;">1</div>
                    <span>Holiday Start</span>
                </div>
                <div class="legend-item">
                    <div class="legend-color" style="background: white; color: #059669; font-weight: bold; text-decoration: underline; display: flex; align-items: center; justify-content: center; font-size: 12px;">15</div>
                    <span>Holiday Period</span>
                </div>
                <div class="legend-item">
                    <div class="legend-color" style="background: white; border: 2px solid #ef4444; color: #ef4444; font-weight: bold; display: flex; align-items: center; justify-content: center; font-size: 12px;">31</div>
                    <span>Holiday End</span>
                </div>
            </div>
            
            <div class="holiday-info" id="holiday-info">
                <h3>Select your holiday start date</h3>
                <p>Click on any date to automatically calculate 20 working days of holiday</p>
            </div>
        </div>
    </div>

    <script>
        // Australian public holidays for Dec 2025 and Jan 2026
        const publicHolidays = {
            '2025-12-25': 'Christmas Day',
            '2025-12-26': 'Boxing Day',
            '2026-01-01': "New Year's Day",
            '2026-01-26': 'Australia Day'
        };

        let selectedStartDate = null;
        let selectedEndDate = null;
        let holidayRange = [];

        // Generate calendar for a specific month
        function generateCalendar(year, month, containerId) {
            const container = document.getElementById(containerId);
            const firstDay = new Date(year, month, 1);
            const lastDay = new Date(year, month + 1, 0);
            const daysInMonth = lastDay.getDate();
            const startingDay = firstDay.getDay();

            // Clear container
            container.innerHTML = '';

            // Add day headers
            const dayHeaders = ['Sun', 'Mon', 'Tue', 'Wed', 'Thu', 'Fri', 'Sat'];
            dayHeaders.forEach(day => {
                const dayHeader = document.createElement('div');
                dayHeader.className = 'day-header';
                dayHeader.textContent = day;
                container.appendChild(dayHeader);
            });

            // Add empty cells for days before month starts
            for (let i = 0; i < startingDay; i++) {
                const emptyCell = document.createElement('div');
                emptyCell.className = 'day-cell other-month';
                const prevMonthDate = new Date(year, month, 0 - startingDay + i + 1).getDate();
                emptyCell.textContent = prevMonthDate;
                container.appendChild(emptyCell);
            }

            // Add days of the month
            for (let day = 1; day <= daysInMonth; day++) {
                const dayCell = document.createElement('div');
                dayCell.className = 'day-cell';
                dayCell.textContent = day;

                const currentDate = new Date(year, month, day);
                const dateString = formatDate(currentDate);
                const dayOfWeek = currentDate.getDay();

                // Check if weekend
                if (dayOfWeek === 0 || dayOfWeek === 6) {
                    dayCell.classList.add('weekend');
                }

                // Check if public holiday
                if (publicHolidays[dateString]) {
                    dayCell.classList.add('public-holiday');
                    dayCell.title = publicHolidays[dateString];
                }

                // Add click event
                dayCell.addEventListener('click', () => selectStartDate(currentDate));
                dayCell.dataset.date = dateString;

                container.appendChild(dayCell);
            }

            // Add empty cells for days after month ends
            const totalCells = container.children.length - 7; // Subtract day headers
            const remainingCells = 42 - totalCells; // 6 rows × 7 days = 42 cells
            for (let i = 1; i <= remainingCells && totalCells + i <= 35; i++) {
                const emptyCell = document.createElement('div');
                emptyCell.className = 'day-cell other-month';
                emptyCell.textContent = i;
                container.appendChild(emptyCell);
            }
        }

        // Format date as YYYY-MM-DD
        function formatDate(date) {
            return date.getFullYear() + '-' + 
                   String(date.getMonth() + 1).padStart(2, '0') + '-' + 
                   String(date.getDate()).padStart(2, '0');
        }

        // Check if date is a working day (not weekend or public holiday)
        function isWorkingDay(date) {
            const dayOfWeek = date.getDay();
            const dateString = formatDate(date);
            
            // Weekend check
            if (dayOfWeek === 0 || dayOfWeek === 6) return false;
            
            // Public holiday check
            if (publicHolidays[dateString]) return false;
            
            return true;
        }

        // Calculate end date after 20 working days
        function calculateEndDate(startDate) {
            let workingDays = 0;
            let currentDate = new Date(startDate);
            
            while (workingDays < 20) {
                if (isWorkingDay(currentDate)) {
                    workingDays++;
                }
                if (workingDays < 20) {
                    currentDate.setDate(currentDate.getDate() + 1);
                }
            }
            
            return currentDate;
        }

        // Get all dates in range
        function getDatesInRange(startDate, endDate) {
            const dates = [];
            let currentDate = new Date(startDate);
            
            while (currentDate <= endDate) {
                dates.push(new Date(currentDate));
                currentDate.setDate(currentDate.getDate() + 1);
            }
            
            return dates;
        }

        // Clear previous selection
        function clearSelection() {
            document.querySelectorAll('.day-cell').forEach(cell => {
                cell.classList.remove('selected-start', 'selected-end', 'selected-range');
            });
        }

        // Highlight holiday range
        function highlightRange(startDate, endDate, rangeDates) {
            clearSelection();
            
            const startDateString = formatDate(startDate);
            const endDateString = formatDate(endDate);
            
            rangeDates.forEach(date => {
                const dateString = formatDate(date);
                const cell = document.querySelector(`[data-date="${dateString}"]`);
                
                if (cell) {
                    if (dateString === startDateString) {
                        cell.classList.add('selected-start');
                    } else if (dateString === endDateString) {
                        cell.classList.add('selected-end');
                    } else {
                        cell.classList.add('selected-range');
                    }
                }
            });
        }

        // Select start date and calculate holiday
        function selectStartDate(date) {
            selectedStartDate = new Date(date);
            selectedEndDate = calculateEndDate(selectedStartDate);
            holidayRange = getDatesInRange(selectedStartDate, selectedEndDate);
            
            highlightRange(selectedStartDate, selectedEndDate, holidayRange);
            updateHolidayInfo();
        }

        // Update holiday information display
        function updateHolidayInfo() {
            const infoPanel = document.getElementById('holiday-info');
            
            if (!selectedStartDate || !selectedEndDate) {
                infoPanel.innerHTML = `
                    <h3>Select your holiday start date</h3>
                    <p>Click on any date to automatically calculate 20 working days of holiday</p>
                `;
                return;
            }

            const totalDays = holidayRange.length;
            const workingDays = holidayRange.filter(date => isWorkingDay(date)).length;
            const weekendDays = holidayRange.filter(date => {
                const dayOfWeek = date.getDay();
                return dayOfWeek === 0 || dayOfWeek === 6;
            }).length;
            const publicHolidayDays = holidayRange.filter(date => {
                const dateString = formatDate(date);
                return publicHolidays[dateString];
            }).length;

            const formatDisplayDate = (date) => {
                return date.toLocaleDateString('en-AU', {
                    weekday: 'long',
                    year: 'numeric',
                    month: 'long',
                    day: 'numeric'
                });
            };

            infoPanel.innerHTML = `
                <h3>🎉 Your Holiday Plan</h3>
                <div class="holiday-details">
                    <div class="detail-item">
                        <div class="detail-label">Start Date:</div>
                        <div>${formatDisplayDate(selectedStartDate)}</div>
                    </div>
                    <div class="detail-item">
                        <div class="detail-label">End Date:</div>
                        <div>${formatDisplayDate(selectedEndDate)}</div>
                    </div>
                    <div class="detail-item">
                        <div class="detail-label">Total Calendar Days:</div>
                        <div>${totalDays} days</div>
                    </div>
                    <div class="detail-item">
                        <div class="detail-label">Working Days:</div>
                        <div>${workingDays} days</div>
                    </div>
                    <div class="detail-item">
                        <div class="detail-label">Weekend Days:</div>
                        <div>${weekendDays} days</div>
                    </div>
                    <div class="detail-item">
                        <div class="detail-label">Public Holidays:</div>
                        <div>${publicHolidayDays} days</div>
                    </div>
                </div>
            `;
        }

        // Initialize calendars
        document.addEventListener('DOMContentLoaded', () => {
            generateCalendar(2025, 11, 'december-calendar'); // December is month 11
            generateCalendar(2026, 0, 'january-calendar');   // January is month 0
        });
    </script>
</body>
</html>


~~~

"""
