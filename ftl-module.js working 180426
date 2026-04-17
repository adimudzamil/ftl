// ftl-module.js - FTL Calculation Module for Roster Parser

// Global configuration
let ftlConfig = {
    stations: {},
    aircraftGroups: {},
    limits: {}
};

let currentFtlSettings = {
    crewType: 'tech',
    acclimatization: 'acclimatized'
};

// Load all configuration files
async function loadFtlConfiguration() {
    try {
        console.log('Loading FTL configuration files...');
        
        const [stations, aircraftGroups, limits] = await Promise.all([
            fetch('stations.json').then(r => r.json()),
            fetch('aircraft-groups.json').then(r => r.json()),
            fetch('ftl-limits.json').then(r => r.json())
        ]);
        
        ftlConfig.stations = stations;
        ftlConfig.aircraftGroups = aircraftGroups;
        ftlConfig.limits = limits;
        
        console.log('FTL configuration loaded successfully:', {
            stations: Object.keys(stations).length,
            aircraftGroups: Object.keys(aircraftGroups).length,
            limits: Object.keys(limits).length
        });
        
        document.getElementById('ftl-status').textContent = '✓ FTL data loaded';
        document.getElementById('ftl-status').style.color = '#28a745';
        
        return true;
    } catch (error) {
        console.error('Failed to load FTL configuration:', error);
        document.getElementById('ftl-status').textContent = '✗ FTL data failed to load';
        document.getElementById('ftl-status').style.color = '#dc3545';
        return false;
    }
}

// Get time band from local time (HH:MM format)
function getLocalStartTimeBand(localTimeStr) {
    const hour = parseInt(localTimeStr.split(':')[0]);
    const minute = parseInt(localTimeStr.split(':')[1]);
    const totalMinutes = hour * 60 + minute;
    
    if (totalMinutes >= 6*60 && totalMinutes < 8*60) return '0600-0759';
    if (totalMinutes >= 8*60 && totalMinutes < 13*60) return '0800-1259';
    if (totalMinutes >= 13*60 && totalMinutes < 18*60) return '1300-1759';
    if (totalMinutes >= 18*60 && totalMinutes < 22*60) return '1800-2159';
    return '2200-0559';
}

// Get FTL limit in hours based on duty start time and sector count
function getFtlLimitHours(dutyStartTime, sectorCount, restHours = null) {
    const { crewType, acclimatization } = currentFtlSettings;
    const timeBand = getLocalStartTimeBand(dutyStartTime);
    
    let limitTable;
    let ruleType = '';
    
    if (acclimatization === 'acclimatized') {
        limitTable = ftlConfig.limits.acclimatized[crewType][timeBand];
        ruleType = 'acclimatized';
    } else {
        // Non-acclimatized logic - determine which rest rule applies
        if (restHours !== null) {
            if (restHours <= 18 || restHours >= 30) {
                ruleType = 'rest_period_leq18_or_geq30';
            } else {
                ruleType = 'rest_period_18_01_to_29_59';
            }
        } else {
            // Default to more restrictive rule
            ruleType = 'rest_period_leq18_or_geq30';
        }
        
        limitTable = ftlConfig.limits.non_acclimatized[ruleType][crewType];
        // Non-acclimatized only covers up to 4 sectors
        sectorCount = Math.min(sectorCount, 4);
    }
    
    if (!limitTable || limitTable.length === 0) {
        console.warn(`No limit table found for ${crewType}, ${acclimatization}, ${ruleType || timeBand}`);
        return null;
    }
    
    // SectorCount is 1-indexed for the array (1 sector = index 0)
    const sectorIndex = Math.min(sectorCount, limitTable.length) - 1;
    const limitHours = limitTable[sectorIndex];
    
    console.log(`FTL Limit: ${limitHours}h for ${sectorCount} sectors, ${acclimatization}, ${ruleType || timeBand}, ${dutyStartTime}`);
    
    return limitHours;
}

// Calculate latest arrival time based on duty start and FTL limit
function calculateLatestArrival(dutyStartTime, ftlLimitHours, aircraftCode, destinationGMT) {
    // Convert "HH:MM" to total minutes
    const [startHour, startMin] = dutyStartTime.split(':').map(Number);
    let startTotalMinutes = (startHour * 60) + startMin;
    
    // Calculate latest duty end in minutes
    const limitTotalMinutes = ftlLimitHours * 60;
    let latestDutyEndMinutes = startTotalMinutes + limitTotalMinutes;
    
    // Determine post-flight duty time (minutes)
    let postFlightTime = 30; // Default for Cabin Crew and B737 Tech
    if (currentFtlSettings.crewType === 'tech' && aircraftCode) {
        const acType = aircraftCode.substring(0, 3);
        const group = ftlConfig.aircraftGroups[acType];
        postFlightTime = (group === 'Widebody') ? 45 : 30;
    }
    
    // Latest Arrival = Latest Duty End - Post-Flight Time
    let latestArrivalMinutes = latestDutyEndMinutes - postFlightTime;
    
    // Handle rollover past midnight
    while (latestArrivalMinutes < 0) latestArrivalMinutes += 24 * 60;
    while (latestArrivalMinutes >= 24 * 60) latestArrivalMinutes -= 24 * 60;
    
    // Format back to "HH:MM"
    const h = Math.floor(latestArrivalMinutes / 60).toString().padStart(2, '0');
    const m = (latestArrivalMinutes % 60).toString().padStart(2, '0');
    
    return `${h}:${m}`;
}

// Calculate rest hours between two dates
function calculateRestHours(previousDutyEnd, currentDutyStart) {
    if (!previousDutyEnd || !currentDutyStart) return null;
    
    const endTime = new Date(previousDutyEnd);
    const startTime = new Date(currentDutyStart);
    
    const diffMs = startTime - endTime;
    const diffHours = diffMs / (1000 * 60 * 60);
    
    return diffHours;
}

// Main function to calculate FTL for all duty cycles
function calculateFTLForFlights(flights) {
    console.log('Calculating FTL for', flights.length, 'flights');
    
    // Group flights into duty cycles
    let dutyCycleFlights = [];
    let dutyStartFlight = null;
    let previousFlight = null;
    
    for (let i = 0; i < flights.length; i++) {
        const flight = flights[i];
        
        // If this flight has dutyStart, start a new duty cycle
        if (flight.dutyStart) {
            dutyStartFlight = flight;
            dutyCycleFlights = [flight];
        } 
        // If it's part of an ongoing duty cycle
        else if (dutyStartFlight) {
            dutyCycleFlights.push(flight);
        }
        
        // If this flight ends a duty cycle
        if (flight.dutyEnd && dutyStartFlight && dutyCycleFlights.length > 0) {
            const sectorCount = dutyCycleFlights.length;
            const lastFlight = dutyCycleFlights[dutyCycleFlights.length - 1];
            
            // Calculate rest hours from previous duty if available
            let restHours = null;
            if (previousFlight && previousFlight.dutyEnd) {
                restHours = calculateRestHours(previousFlight.dutyEnd, dutyStartFlight.dutyStart);
            }
            
            // Get FTL limit
            const ftlLimitHours = getFtlLimitHours(
                dutyStartFlight.dutyStart.split(' ')[1], // Just the time part
                sectorCount,
                restHours
            );
            
            if (ftlLimitHours) {
                // Calculate latest arrival
                const latestArrival = calculateLatestArrival(
                    dutyStartFlight.dutyStart.split(' ')[1],
                    ftlLimitHours,
                    lastFlight.aircraft,
                    lastFlight.destinationGMT
                );
                
                // Store FTL data on the last flight of the cycle
                lastFlight.ftlData = {
                    limitHours: ftlLimitHours,
                    latestArrivalTime: latestArrival,
                    sectorCount: sectorCount,
                    dutyStartTime: dutyStartFlight.dutyStart,
                    calculatedAt: new Date().toISOString()
                };
                
                console.log(`Duty cycle ${i}: ${sectorCount} sectors, FTL limit: ${ftlLimitHours}h, Latest arrival: ${latestArrival}`);
            }
            
            // Reset for next cycle
            dutyStartFlight = null;
            dutyCycleFlights = [];
        }
        
        previousFlight = flight;
    }
    
    return flights;
}

// Update table headers to include FTL columns
function updateTableHeaders() {
    const tableHeaders = document.querySelector('#results table thead tr');
    if (tableHeaders && !tableHeaders.innerHTML.includes('FTL Limit')) {
        tableHeaders.innerHTML += `
            <th>FTL Limit (hrs)</th>
            <th>Latest Arrival</th>
            <th>Sectors</th>
        `;
    }
}

// Update table rows with FTL data
function updateTableRows(flights) {
    const rows = document.querySelectorAll('#results table tbody tr');
    
    rows.forEach((row, index) => {
        if (index < flights.length) {
            const flight = flights[index];
            
            // Add FTL cells if they don't exist
            if (!row.querySelector('.ftl-limit')) {
                row.innerHTML += `
                    <td class="ftl-limit">${flight.ftlData ? flight.ftlData.limitHours.toFixed(2) : ''}</td>
                    <td class="ftl-latest">${flight.ftlData ? flight.ftlData.latestArrivalTime : ''}</td>
                    <td class="ftl-sectors">${flight.ftlData ? flight.ftlData.sectorCount : ''}</td>
                `;
            } else {
                // Update existing cells
                row.querySelector('.ftl-limit').textContent = flight.ftlData ? flight.ftlData.limitHours.toFixed(2) : '';
                row.querySelector('.ftl-latest').textContent = flight.ftlData ? flight.ftlData.latestArrivalTime : '';
                row.querySelector('.ftl-sectors').textContent = flight.ftlData ? flight.ftlData.sectorCount : '';
            }
            
            // Add highlighting if FTL data exists
            if (flight.ftlData) {
                row.style.backgroundColor = '#f8f9fa';
            }
        }
    });
}

// Main function to load and calculate FTL
async function loadAndCalculateFTL() {
    // Update current settings
    currentFtlSettings.crewType = document.getElementById('crewTypeSelect').value;
    currentFtlSettings.acclimatization = document.getElementById('acclimatizationSelect').value;
    
    console.log('FTL Settings updated:', currentFtlSettings);
    
    // Load configuration if not already loaded
    if (!ftlConfig.stations || Object.keys(ftlConfig.stations).length === 0) {
        const loaded = await loadFtlConfiguration();
        if (!loaded) {
            alert('Failed to load FTL configuration. Please check the console for errors.');
            return;
        }
    }
    
    // Trigger a re-parse of the roster if flights are already parsed
    const currentInput = document.getElementById('rosterInput').value;
    if (currentInput) {
        // Check if we have a parseRoster function
        if (typeof parseRoster === 'function') {
            parseRoster();
        } else {
            alert('Please parse the roster first using the main parser.');
        }
    } else {
        alert('Please paste a roster first.');
    }
}

// Initialize when page loads
document.addEventListener('DOMContentLoaded', async function() {
    // Load FTL configuration
    await loadFtlConfiguration();
    
    // Add event listeners for settings changes
    document.getElementById('crewTypeSelect').addEventListener('change', function() {
        currentFtlSettings.crewType = this.value;
        console.log('Crew type changed to:', this.value);
    });
    
    document.getElementById('acclimatizationSelect').addEventListener('change', function() {
        currentFtlSettings.acclimatization = this.value;
        console.log('Acclimatization changed to:', this.value);
    });
});