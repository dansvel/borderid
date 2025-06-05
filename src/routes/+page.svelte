<script>
  let provinces = $state([]);
  let districts = $state([]);
  let subDistricts = $state([]);
  let villages = $state([]);
  
  let selectedProvince = $state('');
  let selectedDistrict = $state('');
  let selectedSubDistrict = $state('');
  let selectedVillage = $state('');
  
  let allData = $state([]);
  let activeTab = $state('geojson');
  let filteredData = $state([]);
  let isLoading = $state(false);
  let showResults = $state(false);

  async function loadData() {
    isLoading = true;
    try {
      const response = await fetch('https://drive.indramayukab.my.id/indonesia_villages_border.geojson');
      allData = await response.json();
      
      if (Array.isArray(allData)) {
        provinces = [...new Set(allData.map(item => item.province))].sort((a, b) => a.localeCompare(b, 'id'));
      }
    } catch (error) {
      console.error('Error loading data:', error);
    }
    isLoading = false;
  }

  async function updateDistricts() {
    isLoading = true;
    if (selectedProvince) {
      districts = [...new Set(allData
        .filter(item => item.province === selectedProvince)
        .map(item => item.district))]
        .sort((a, b) => a.localeCompare(b, 'id'));
    } else {
      districts = [];
    }
    selectedDistrict = '';
    selectedSubDistrict = '';
    selectedVillage = '';
    await updateSubDistricts();
    await updateFilteredData();
    isLoading = false;
  }

  async function updateSubDistricts() {
    isLoading = true;
    if (selectedDistrict) {
      subDistricts = [...new Set(allData
        .filter(item => item.province === selectedProvince && item.district === selectedDistrict)
        .map(item => item.sub_district))]
        .sort((a, b) => a.localeCompare(b, 'id'));
    } else {
      subDistricts = [];
    }
    selectedSubDistrict = '';
    selectedVillage = '';
    await updateVillages();
    await updateFilteredData();
    isLoading = false;
  }

  async function updateVillages() {
    isLoading = true;
    if (selectedSubDistrict) {
      villages = [...new Set(allData
        .filter(item => 
          item.province === selectedProvince && 
          item.district === selectedDistrict && 
          item.sub_district === selectedSubDistrict)
        .map(item => item.village))]
        .sort((a, b) => a.localeCompare(b, 'id'));
    } else {
      villages = [];
    }
    selectedVillage = '';
    await updateFilteredData();
    isLoading = false;
  }

  async function updateFilteredData() {
    if (selectedDistrict) {
      filteredData = allData.filter(item => 
        item.province === selectedProvince && 
        item.district === selectedDistrict &&
        (!selectedSubDistrict || item.sub_district === selectedSubDistrict) &&
        (!selectedVillage || item.village === selectedVillage)
      );
    } else {
      filteredData = [];
    }
  }

  function convertToGPX() {
    if (filteredData.length === 0) return '';
    
    let gpx = `<?xml version="1.0" encoding="UTF-8"?>
<gpx version="1.1" creator="BorderID Converter"
     xmlns="http://www.topografix.com/GPX/1/1"
     xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
     xsi:schemaLocation="http://www.topografix.com/GPX/1/1 http://www.topografix.com/GPX/1/1/gpx.xsd">
  <metadata>
    <name>${getLocationName()}</name>
    <time>${new Date().toISOString()}</time>
  </metadata>`;

    filteredData.forEach(item => {
      const name = `${item.province} - ${item.district} - ${item.sub_district} - ${item.village}`;
      gpx += `
  <trk>
    <name>${name}</name>
    <trkseg>`;
      
      item.border.forEach(coord => {
        gpx += `
      <trkpt lat="${coord[1]}" lon="${coord[0]}">
        <ele>0</ele>
      </trkpt>`;
      });

      gpx += `
    </trkseg>
  </trk>`;
    });

    gpx += `
</gpx>`;

    return gpx;
  }

  function getLocationName() {
    let name = `${selectedProvince} - ${selectedDistrict}`;
    if (selectedSubDistrict) name += ` - ${selectedSubDistrict}`;
    if (selectedVillage) name += ` - ${selectedVillage}`;
    return name;
  }

  function downloadData() {
    if (filteredData.length === 0) return;
    
    let dataStr, dataUri, exportFileDefaultName;
    
    if (activeTab === 'geojson') {
      dataStr = JSON.stringify(filteredData, null, 2);
      dataUri = 'data:application/json;charset=utf-8,'+ encodeURIComponent(dataStr);
      exportFileDefaultName = `${getLocationName().replace(/ - /g, '_')}.geojson`;
    } else {
      dataStr = convertToGPX();
      dataUri = 'data:application/gpx+xml;charset=utf-8,'+ encodeURIComponent(dataStr);
      exportFileDefaultName = `${getLocationName().replace(/ - /g, '_')}.gpx`;
    }
    
    const linkElement = document.createElement('a');
    linkElement.setAttribute('href', dataUri);
    linkElement.setAttribute('download', exportFileDefaultName);
    linkElement.click();
  }

  loadData();
</script>

<div>
  <h1>Pilih Lokasi</h1>
  
  <div>
    <label for="province">Provinsi:</label>
    <select 
      id="province"
      bind:value={selectedProvince} 
      on:change={updateDistricts}
    >
      <option value="">-- Pilih Provinsi --</option>
      {#each provinces as province}
        <option value={province}>{province}</option>
      {/each}
    </select>
  </div>

  <div>
    <label for="district">Kabupaten/Kota:</label>
    <select 
      id="district"
      bind:value={selectedDistrict} 
      on:change={updateSubDistricts}
      disabled={!selectedProvince || isLoading}
    >
      <option value="">-- Pilih Kabupaten/Kota --</option>
      {#each districts as district}
        <option value={district}>{district}</option>
      {/each}
    </select>
  </div>

  <div>
    <label for="subDistrict">Kecamatan:</label>
    <select 
      id="subDistrict"
      bind:value={selectedSubDistrict} 
      on:change={updateVillages}
      disabled={!selectedDistrict || isLoading}
    >
      <option value="">-- Pilih Kecamatan --</option>
      {#each subDistricts as subDistrict}
        <option value={subDistrict}>{subDistrict}</option>
      {/each}
    </select>
  </div>

  <div>
    <label for="village">Desa/Kelurahan:</label>
    <select 
      id="village"
      bind:value={selectedVillage}
      on:change={updateFilteredData}
      disabled={!selectedSubDistrict || isLoading}
    >
      <option value="">-- Pilih Desa/Kelurahan --</option>
      {#each villages as village}
        <option value={village}>{village}</option>
      {/each}
    </select>
  </div>

  {#if selectedDistrict}
    <div class="selected-info">
      <p>
        Anda memilih: {getLocationName()}
      </p>
      <p>
        Jumlah data: {filteredData.length} {filteredData.length > 1 ? 'lokasi' : 'lokasi'}
      </p>
      <div class="button-group">
        <button 
          class="tab-btn {activeTab === 'geojson' ? 'active' : ''}" 
          on:click={() => activeTab = 'geojson'}
          disabled={isLoading}
        >
          GeoJSON
        </button>
        <button 
          class="tab-btn {activeTab === 'gpx' ? 'active' : ''}" 
          on:click={() => activeTab = 'gpx'}
          disabled={isLoading}
        >
          GPX
        </button>
        <button on:click={downloadData} class="download-btn" disabled={isLoading}>
          Download
        </button>
        <button 
          on:click={() => showResults = !showResults} 
          class="toggle-btn"
          disabled={isLoading}
        >
          {showResults ? 'Sembunyikan Hasil' : 'Tampilkan Hasil'}
        </button>
      </div>
    </div>
  {/if}

  {#if filteredData.length > 0 && showResults}
    <div class="data-preview">
      <h2>Data {activeTab.toUpperCase()}:</h2>
      <pre>
        {#if activeTab === 'geojson'}
          {JSON.stringify(filteredData, null, 2)}
        {:else}
          {convertToGPX()}
        {/if}
      </pre>
    </div>
  {/if}
</div>

{#if isLoading}
  <div class="loading-backdrop">
    <div class="loading-spinner"></div>
  </div>
{/if}

<style>
  div {
    margin-bottom: 1rem;
  }
  
  label {
    display: inline-block;
    width: 150px;
  }
  
  select {
    width: 300px;
    padding: 0.5rem;
  }
  
  .selected-info {
    margin-top: 2rem;
    padding: 1rem;
    background-color: #f0f0f0;
    border-radius: 4px;
  }

  .button-group {
    display: flex;
    gap: 1rem;
    margin-top: 1rem;
  }

  .download-btn, .toggle-btn {
    padding: 0.5rem 1rem;
    color: white;
    border: none;
    border-radius: 4px;
    cursor: pointer;
  }

  .download-btn {
    background-color: #4CAF50;
  }

  .toggle-btn {
    background-color: #2196F3;
  }

  .download-btn:hover:not(:disabled),
  .toggle-btn:hover:not(:disabled) {
    opacity: 0.9;
  }

  .download-btn:disabled,
  .toggle-btn:disabled {
    background-color: #cccccc;
    cursor: not-allowed;
  }

  .tab-btn {
    padding: 0.5rem 1rem;
    background-color: #f0f0f0;
    border: 1px solid #ddd;
    border-radius: 4px;
    cursor: pointer;
  }

  .tab-btn.active {
    background-color: #4CAF50;
    color: white;
    border-color: #4CAF50;
  }

  .tab-btn:disabled {
    background-color: #cccccc;
    cursor: not-allowed;
  }

  .data-preview {
    margin-top: 1rem;
    padding: 1rem;
    background-color: #f8f9fa;
    border-radius: 4px;
    overflow-x: auto;
  }

  .data-preview pre {
    white-space: pre-wrap;
    word-wrap: break-word;
    font-family: monospace;
    font-size: 0.9rem;
    line-height: 1.4;
  }

  .loading-backdrop {
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background-color: rgba(0, 0, 0, 0.5);
    display: flex;
    justify-content: center;
    align-items: center;
    z-index: 1000;
  }

  .loading-spinner {
    width: 50px;
    height: 50px;
    border: 5px solid #f3f3f3;
    border-top: 5px solid #4CAF50;
    border-radius: 50%;
    animation: spin 1s linear infinite;
  }

  @keyframes spin {
    0% { transform: rotate(0deg); }
    100% { transform: rotate(360deg); }
  }
</style>
