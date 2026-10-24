<template>
    <div class="dashboard" @click="handleGlobalClick">
        <!-- 扫描线动效 -->
        <div class="scanline"></div>

        <!-- 顶部导航栏 -->
        <header class="dash-header">
            <div class="header-left">
                <button class="neon-btn" @click.stop="toadd()">
                    <span class="btn-glow"></span>
                    <span class="btn-text">{{ t.addCity }}</span>
                </button>
                <div class="feedback-link" :class="{show: isac}">
                    <a href="https://qun.qq.com/universal-share/share?ac=1&authKey=z92qJdPVwqCnPpnMSKhNRS5PoBn1erS9yAB0GAjmU9XagRlHGSlAI%2FZ%2FRONshIsm&busi_data=eyJncm91cENvZGUiOiIyOTAwNzgzNTMiLCJ0b2tlbiI6Iks4dG1ERlV2NkZvb28raGI2MGdZb2NlY3BZOVFCYUdXYVdFcWhaT2dmZk0yY3BzNFVkdXl5anRXV0NGQTRiTDUiLCJ1aW4iOiIyODU0MzcyOTYyIn0%3D&data=28rJgYEA_Lt0--8PEvuRxcNAiAzRCGIi-fyvEAZ_zn8WyKhQ9k6B0Ss1BZ4TteckYMfdSPpBQ8cQDUCKA-uatw&svctype=4&tempid=h5_group_info" target="_blank" @keyup.enter="addToList">{{ t.feedback }}</a>
                </div>
            </div>
            <div class="header-center">
                <div class="title-wrapper">
                    <div class="title-decoration left"></div>
                    <h1 class="dash-title">{{ t.title }}</h1>
                    <div class="title-decoration right"></div>
                </div>
                <p class="dash-subtitle">{{ t.subtitle }}</p>
            </div>
            <div class="header-right">
                <!-- 语言切换 -->
                <div class="lang-switcher" @click.stop>
                    <button class="lang-btn" @click="langMenuOpen = !langMenuOpen">
                        <span class="lang-icon">🌐</span>
                        <span class="lang-label">{{ langLabels[currentLang] }}</span>
                        <span class="lang-arrow" :class="{open: langMenuOpen}">▾</span>
                    </button>
                    <div class="lang-dropdown" v-if="langMenuOpen">
                        <button
                            v-for="(label, code) in langLabels"
                            :key="code"
                            :class="{active: currentLang === code}"
                            @click="setLang(code)">
                            {{ label }}
                        </button>
                    </div>
                </div>
                <div class="github-badges">
                    <a href="https://github.com/lllomh/Position-real-time-monitoring" target="_blank"><img src="https://img.shields.io/github/stars/lllomh/Position-real-time-monitoring?style=social" alt=""/></a>
                    <a href="https://github.com/lllomh/Position-real-time-monitoring" target="_blank"><img src="https://img.shields.io/github/forks/lllomh/Position-real-time-monitoring?style=social" alt=""/></a>
                    <a href="https://github.com/lllomh/Position-real-time-monitoring" target="_blank"><img src="https://img.shields.io/github/watchers/lllomh/Position-real-time-monitoring?style=social" alt=""/></a>
                </div>
            </div>
        </header>

        <!-- 主体内容 -->
        <main class="dash-body">
            <!-- 左侧：图表 + 卡片 -->
            <section class="dash-left">
                <!-- 图表区域 -->
                <div class="chart-panel glass-card">
                    <div class="panel-header">
                        <div class="panel-title">
                            <span class="dot pulse"></span>
                            {{ t.trendTitle }}
                        </div>
                        <div class="btn-3d-group">
                            <button class="btn-3d" @click="view3DMode = 'bar'" :disabled="!data || !data_rs_more">
                                <span class="btn-3d-cube">⬛</span> {{ t.view3d }}
                            </button>
                            <button class="btn-3d btn-3d-globe" @click="view3DMode = 'globe'" :disabled="!data || !data_rs_more">
                                <span class="btn-3d-cube">🌐</span> {{ t.view3dGlobe }}
                            </button>
                        </div>
                        <div class="city-tabs">
                            <select class="city-select" :value="selectedMoreCity" :aria-label="t.cityCol" :disabled="!moreHistory.length" @change="selectCity($event.target.value)">
                                <option value="" disabled>{{ t.cityCol }}</option>
                                <option v-for="(name, key) in t.citiesRank" :key="key" :value="key">{{ name }}</option>
                            </select>
                            <button :class="{active: selectedCity === 'sh'}" @click="selectCity('sh')">{{ t.cities.sh }}</button>
                            <button :class="{active: selectedCity === 'bj'}" @click="selectCity('bj')">{{ t.cities.bj }}</button>
                            <button :class="{active: selectedCity === 'sz'}" @click="selectCity('sz')">{{ t.cities.sz }}</button>
                            <button :class="{active: selectedCity === 'gz'}" @click="selectCity('gz')">{{ t.cities.gz }}</button>
                        </div>
                    </div>
                    <div class="year-tabs" v-if="availableYears.length > 0">
                        <button :class="{active: selectedYear === 'all'}" @click="setYear('all')">{{ t.all }}</button>
                        <button v-for="y in availableYears" :key="y" :class="{active: selectedYear === y}" @click="setYear(y)">{{y}}</button>
                    </div>
                    <div class="chart-container">
                        <div class="chart-loading" v-if="loading">
                            <div class="chart-spinner"></div>
                            <p class="chart-loading-text">{{ t.loading }}</p>
                        </div>
                        <div id="main" style="width: 100%; height: 100%"></div>
                    </div>
                </div>

                <!-- 城市数据卡片 -->
                <div class="city-cards">
                    <!-- 信息卡 -->
                    <div class="stat-card glass-card info-card">
                        <div class="card-corner tl"></div>
                        <div class="card-corner tr"></div>
                        <div class="card-corner bl"></div>
                        <div class="card-corner br"></div>
                        <div class="card-label">
                            <template v-if="data">{{data[(data.length-1)].ceate_time}}</template>
                            <span v-else class="mini-spinner"></span>
                        </div>
                        <div class="card-hint">{{ t.dataUpdateHint }}</div>
                        <div class="card-divider"></div>
                        <div class="card-row">
                            <span class="framework-tag react">React</span>
                            <span class="card-value dim">--</span>
                        </div>
                        <div class="card-row">
                            <span class="framework-tag vue">Vue</span>
                            <span class="card-value dim">--</span>
                        </div>
                    </div>

                    <!-- 上海 -->
                    <div class="stat-card glass-card">
                        <div class="card-corner tl"></div>
                        <div class="card-corner tr"></div>
                        <div class="card-corner bl"></div>
                        <div class="card-corner br"></div>
                        <div class="card-label">{{ t.cities.sh }}</div>
                        <div class="card-divider"></div>
                        <div class="card-row">
                            <span class="framework-tag react">R</span>
                            <span class="card-value react-val">
                                <NumberGrow v-if="data" :value="strs((JSON.parse(data[(data.length-1)].react)['sh']).toString())"></NumberGrow><span v-if="data && strs((JSON.parse(data[(data.length-1)].react)['sh']).toString()) >= 1000" class="plus-sign">+</span>
                                <span v-if="!data" class="mini-spinner"></span>
                            </span>
                        </div>
                        <div class="card-yesterday">{{ t.yesterday }}: <span v-if="data">{{JSON.parse(data[(data.length-2)].react)['sh']}}</span><span v-else class="mini-spinner"></span></div>
                        <div class="card-row">
                            <span class="framework-tag vue">V</span>
                            <span class="card-value vue-val">
                                <NumberGrow v-if="data" :value="strs((JSON.parse(data[(data.length-1)].vue)['sh']).toString())"></NumberGrow><span v-if="data && strs((JSON.parse(data[(data.length-1)].vue)['sh']).toString()) >= 1000" class="plus-sign">+</span>
                                <span v-if="!data" class="mini-spinner"></span>
                            </span>
                        </div>
                        <div class="card-yesterday">{{ t.yesterday }}: <span v-if="data">{{JSON.parse(data[(data.length-2)].vue)['sh']}}</span><span v-else class="mini-spinner"></span></div>
                    </div>

                    <!-- 北京 -->
                    <div class="stat-card glass-card">
                        <div class="card-corner tl"></div>
                        <div class="card-corner tr"></div>
                        <div class="card-corner bl"></div>
                        <div class="card-corner br"></div>
                        <div class="card-label">{{ t.cities.bj }}</div>
                        <div class="card-divider"></div>
                        <div class="card-row">
                            <span class="framework-tag react">R</span>
                            <span class="card-value react-val">
                                <NumberGrow v-if="data" :value="strs((JSON.parse(data[(data.length-1)].react)['bj']).toString())"></NumberGrow><span v-if="data && strs((JSON.parse(data[(data.length-1)].react)['bj']).toString()) >= 1000" class="plus-sign">+</span>
                                <span v-if="!data" class="mini-spinner"></span>
                            </span>
                        </div>
                        <div class="card-yesterday">{{ t.yesterday }}: <span v-if="data">{{JSON.parse(data[(data.length-2)].react)['bj']}}</span><span v-else class="mini-spinner"></span></div>
                        <div class="card-row">
                            <span class="framework-tag vue">V</span>
                            <span class="card-value vue-val">
                                <NumberGrow v-if="data" :value="strs((JSON.parse(data[(data.length-1)].vue)['bj']).toString())"></NumberGrow><span v-if="data && strs((JSON.parse(data[(data.length-1)].vue)['bj']).toString()) >= 1000" class="plus-sign">+</span>
                                <span v-if="!data" class="mini-spinner"></span>
                            </span>
                        </div>
                        <div class="card-yesterday">{{ t.yesterday }}: <span v-if="data">{{JSON.parse(data[(data.length-2)].vue)['bj']}}</span><span v-else class="mini-spinner"></span></div>
                    </div>

                    <!-- 深圳 -->
                    <div class="stat-card glass-card">
                        <div class="card-corner tl"></div>
                        <div class="card-corner tr"></div>
                        <div class="card-corner bl"></div>
                        <div class="card-corner br"></div>
                        <div class="card-label">{{ t.cities.sz }}</div>
                        <div class="card-divider"></div>
                        <div class="card-row">
                            <span class="framework-tag react">R</span>
                            <span class="card-value react-val">
                                <NumberGrow v-if="data" :value="strs((JSON.parse(data[(data.length-1)].react)['sz']).toString())"></NumberGrow><span v-if="data && strs((JSON.parse(data[(data.length-1)].react)['sz']).toString()) >= 1000" class="plus-sign">+</span>
                                <span v-if="!data" class="mini-spinner"></span>
                            </span>
                        </div>
                        <div class="card-yesterday">{{ t.yesterday }}: <span v-if="data">{{JSON.parse(data[(data.length-2)].react)['sz']}}</span><span v-else class="mini-spinner"></span></div>
                        <div class="card-row">
                            <span class="framework-tag vue">V</span>
                            <span class="card-value vue-val">
                                <NumberGrow v-if="data" :value="strs((JSON.parse(data[(data.length-1)].vue)['sz']).toString())"></NumberGrow><span v-if="data && strs((JSON.parse(data[(data.length-1)].vue)['sz']).toString()) >= 1000" class="plus-sign">+</span>
                                <span v-if="!data" class="mini-spinner"></span>
                            </span>
                        </div>
                        <div class="card-yesterday">{{ t.yesterday }}: <span v-if="data">{{JSON.parse(data[(data.length-2)].vue)['sz']}}</span><span v-else class="mini-spinner"></span></div>
                    </div>

                    <!-- 广州 -->
                    <div class="stat-card glass-card">
                        <div class="card-corner tl"></div>
                        <div class="card-corner tr"></div>
                        <div class="card-corner bl"></div>
                        <div class="card-corner br"></div>
                        <div class="card-label">{{ t.cities.gz }}</div>
                        <div class="card-divider"></div>
                        <div class="card-row">
                            <span class="framework-tag react">R</span>
                            <span class="card-value react-val">
                                <NumberGrow v-if="data" :value="strs((JSON.parse(data[(data.length-1)].react)['gz']).toString())"></NumberGrow><span v-if="data && strs((JSON.parse(data[(data.length-1)].react)['gz']).toString()) >= 1000" class="plus-sign">+</span>
                                <span v-if="!data" class="mini-spinner"></span>
                            </span>
                        </div>
                        <div class="card-yesterday">{{ t.yesterday }}: <span v-if="data">{{JSON.parse(data[(data.length-2)].react)['gz']}}</span><span v-else class="mini-spinner"></span></div>
                        <div class="card-row">
                            <span class="framework-tag vue">V</span>
                            <span class="card-value vue-val">
                                <NumberGrow v-if="data" :value="strs((JSON.parse(data[(data.length-1)].vue)['gz']).toString())"></NumberGrow><span v-if="data && strs((JSON.parse(data[(data.length-1)].vue)['gz']).toString()) >= 1000" class="plus-sign">+</span>
                                <span v-if="!data" class="mini-spinner"></span>
                            </span>
                        </div>
                        <div class="card-yesterday">{{ t.yesterday }}: <span v-if="data">{{JSON.parse(data[(data.length-2)].vue)['gz']}}</span><span v-else class="mini-spinner"></span></div>
                    </div>
                </div>
            </section>

            <!-- 右侧：城市排行表 -->
            <aside class="dash-right" :class="{ collapsed: tableCollapsed }">
                <div class="table-panel glass-card">
                    <div class="panel-header">
                        <div class="panel-title">
                            <span class="dot pulse cyan"></span>
                            {{ t.rankTitle }}
                        </div>
                        <button class="table-toggle-btn" @click="toggleTable" :title="tableCollapsed ? t.expandTable : t.collapseTable">
                            {{ tableCollapsed ? '▶' : '◀' }}
                        </button>
                    </div>
                    <div class="table-scroll">
                        <table class="rank-table">
                            <thead>
                                <tr>
                                    <th>{{ t.cityCol }}</th>
                                    <th><span class="th-react">React</span></th>
                                    <th><span class="th-vue">Vue</span></th>
                                </tr>
                            </thead>
                            <tbody>
                                <tr>
                                    <td><span class="city-name">{{ t.citiesRank.cq }}</span></td>
                                    <td><NumberGrows v-if="data_rs_more" :value="strs((JSON.parse(data_rs_more.react)['cq']).toString())"></NumberGrows><span v-else class="mini-spinner"></span></td>
                                    <td><NumberGrows v-if="data_rs_more" :value="strs((JSON.parse(data_rs_more.vue)['cq']).toString())"></NumberGrows><span v-else class="mini-spinner"></span></td>
                                </tr>
                                <tr>
                                    <td><span class="city-name">{{ t.citiesRank.gzgz }}</span></td>
                                    <td><NumberGrows v-if="data_rs_more" :value="strs((JSON.parse(data_rs_more.react)['gzgz']).toString())"></NumberGrows><span v-else class="mini-spinner"></span></td>
                                    <td><NumberGrows v-if="data_rs_more" :value="strs((JSON.parse(data_rs_more.vue)['gzgz']).toString())"></NumberGrows><span v-else class="mini-spinner"></span></td>
                                </tr>
                                <tr>
                                    <td><span class="city-name">{{ t.citiesRank.hz }}</span></td>
                                    <td><NumberGrows v-if="data_rs_more" :value="strs((JSON.parse(data_rs_more.react)['hz']).toString())"></NumberGrows><span v-else class="mini-spinner"></span></td>
                                    <td><NumberGrows v-if="data_rs_more" :value="strs((JSON.parse(data_rs_more.vue)['hz']).toString())"></NumberGrows><span v-else class="mini-spinner"></span></td>
                                </tr>
                                <tr>
                                    <td><span class="city-name">{{ t.citiesRank.cd }}</span></td>
                                    <td><NumberGrows v-if="data_rs_more" :value="strs((JSON.parse(data_rs_more.react)['cd']).toString())"></NumberGrows><span v-else class="mini-spinner"></span></td>
                                    <td><NumberGrows v-if="data_rs_more" :value="strs((JSON.parse(data_rs_more.vue)['cd']).toString())"></NumberGrows><span v-else class="mini-spinner"></span></td>
                                </tr>
                                <tr>
                                    <td><span class="city-name">{{ t.citiesRank.tj }}</span></td>
                                    <td><NumberGrows v-if="data_rs_more" :value="strs((JSON.parse(data_rs_more.react)['tj']).toString())"></NumberGrows><span v-else class="mini-spinner"></span></td>
                                    <td><NumberGrows v-if="data_rs_more" :value="strs((JSON.parse(data_rs_more.vue)['tj']).toString())"></NumberGrows><span v-else class="mini-spinner"></span></td>
                                </tr>
                                <tr>
                                    <td><span class="city-name">{{ t.citiesRank.szsz }}</span></td>
                                    <td><NumberGrows v-if="data_rs_more" :value="strs((JSON.parse(data_rs_more.react)['szsz']).toString())"></NumberGrows><span v-else class="mini-spinner"></span></td>
                                    <td><NumberGrows v-if="data_rs_more" :value="strs((JSON.parse(data_rs_more.vue)['szsz']).toString())"></NumberGrows><span v-else class="mini-spinner"></span></td>
                                </tr>
                                <tr>
                                    <td><span class="city-name">{{ t.citiesRank.nj }}</span></td>
                                    <td><NumberGrows v-if="data_rs_more" :value="strs((JSON.parse(data_rs_more.react)['nj']).toString())"></NumberGrows><span v-else class="mini-spinner"></span></td>
                                    <td><NumberGrows v-if="data_rs_more" :value="strs((JSON.parse(data_rs_more.vue)['nj']).toString())"></NumberGrows><span v-else class="mini-spinner"></span></td>
                                </tr>
                                <tr>
                                    <td><span class="city-name">{{ t.citiesRank.gy }}</span></td>
                                    <td><NumberGrows v-if="data_rs_more" :value="strs((JSON.parse(data_rs_more.react)['gy']).toString())"></NumberGrows><span v-else class="mini-spinner"></span></td>
                                    <td><NumberGrows v-if="data_rs_more" :value="strs((JSON.parse(data_rs_more.vue)['gy']).toString())"></NumberGrows><span v-else class="mini-spinner"></span></td>
                                </tr>
                                <tr>
                                    <td><span class="city-name">{{ t.citiesRank.cs }}</span></td>
                                    <td><NumberGrows v-if="data_rs_more" :value="strs((JSON.parse(data_rs_more.react)['cs']).toString())"></NumberGrows><span v-else class="mini-spinner"></span></td>
                                    <td><NumberGrows v-if="data_rs_more" :value="strs((JSON.parse(data_rs_more.vue)['cs']).toString())"></NumberGrows><span v-else class="mini-spinner"></span></td>
                                </tr>
                                <tr>
                                    <td><span class="city-name">{{ t.citiesRank.sy }}</span></td>
                                    <td><NumberGrows v-if="data_rs_more" :value="strs((JSON.parse(data_rs_more.react)['sy']).toString())"></NumberGrows><span v-else class="mini-spinner"></span></td>
                                    <td><NumberGrows v-if="data_rs_more" :value="strs((JSON.parse(data_rs_more.vue)['sy']).toString())"></NumberGrows><span v-else class="mini-spinner"></span></td>
                                </tr>
                                <tr>
                                    <td><span class="city-name">{{ t.citiesRank.jn }}</span></td>
                                    <td><NumberGrows v-if="data_rs_more" :value="strs((JSON.parse(data_rs_more.react)['jn']).toString())"></NumberGrows><span v-else class="mini-spinner"></span></td>
                                    <td><NumberGrows v-if="data_rs_more" :value="strs((JSON.parse(data_rs_more.vue)['jn']).toString())"></NumberGrows><span v-else class="mini-spinner"></span></td>
                                </tr>
                                <tr>
                                    <td><span class="city-name">{{ t.citiesRank.nb }}</span></td>
                                    <td><NumberGrows v-if="data_rs_more" :value="strs((JSON.parse(data_rs_more.react)['nb']).toString())"></NumberGrows><span v-else class="mini-spinner"></span></td>
                                    <td><NumberGrows v-if="data_rs_more" :value="strs((JSON.parse(data_rs_more.vue)['nb']).toString())"></NumberGrows><span v-else class="mini-spinner"></span></td>
                                </tr>
                                <tr>
                                    <td><span class="city-name">{{ t.citiesRank.wx }}</span></td>
                                    <td><NumberGrows v-if="data_rs_more" :value="strs((JSON.parse(data_rs_more.react)['wx']).toString())"></NumberGrows><span v-else class="mini-spinner"></span></td>
                                    <td><NumberGrows v-if="data_rs_more" :value="strs((JSON.parse(data_rs_more.vue)['wx']).toString())"></NumberGrows><span v-else class="mini-spinner"></span></td>
                                </tr>
                                <tr>
                                    <td><span class="city-name">{{ t.citiesRank.cz }}</span></td>
                                    <td><NumberGrows v-if="data_rs_more" :value="strs((JSON.parse(data_rs_more.react)['cz']).toString())"></NumberGrows><span v-else class="mini-spinner"></span></td>
                                    <td><NumberGrows v-if="data_rs_more" :value="strs((JSON.parse(data_rs_more.vue)['cz']).toString())"></NumberGrows><span v-else class="mini-spinner"></span></td>
                                </tr>
                                <tr>
                                    <td><span class="city-name">{{ t.citiesRank.dl }}</span></td>
                                    <td><NumberGrows v-if="data_rs_more" :value="strs((JSON.parse(data_rs_more.react)['dl']).toString())"></NumberGrows><span v-else class="mini-spinner"></span></td>
                                    <td><NumberGrows v-if="data_rs_more" :value="strs((JSON.parse(data_rs_more.vue)['dl']).toString())"></NumberGrows><span v-else class="mini-spinner"></span></td>
                                </tr>
                                <tr>
                                    <td><span class="city-name">{{ t.citiesRank.hf }}</span></td>
                                    <td><NumberGrows v-if="data_rs_more" :value="strs((JSON.parse(data_rs_more.react)['hf']).toString())"></NumberGrows><span v-else class="mini-spinner"></span></td>
                                    <td><NumberGrows v-if="data_rs_more" :value="strs((JSON.parse(data_rs_more.vue)['hf']).toString())"></NumberGrows><span v-else class="mini-spinner"></span></td>
                                </tr>
                                <tr>
                                    <td><span class="city-name">{{ t.citiesRank.wh }}</span></td>
                                    <td><NumberGrows v-if="data_rs_more" :value="strs((JSON.parse(data_rs_more.react)['wh']).toString())"></NumberGrows><span v-else class="mini-spinner"></span></td>
                                    <td><NumberGrows v-if="data_rs_more" :value="strs((JSON.parse(data_rs_more.vue)['wh']).toString())"></NumberGrows><span v-else class="mini-spinner"></span></td>
                                </tr>
                            </tbody>
                        </table>
                    </div>
                </div>
            </aside>
        </main>

        <!-- 3D 视图覆盖层 -->
        <Chart3D
            v-if="view3DMode === 'bar'"
            :cityData="allCityData3D"
            :t="t"
            @close="view3DMode = null"
        />
        <Globe3D
            v-if="view3DMode === 'globe'"
            :cityData="allCityData3D"
            :t="t"
            @close="view3DMode = null"
        />
    </div>
</template>

<script>
    import NumberGrow from '../components/numver'
    import NumberGrows from '../components/numvers'
    import Chart3D from '../components/Chart3D'
    import Globe3D from '../components/Globe3D'
    import echarts from 'echarts'
    import {
        P_GET_DATALIST_MORE ,
        P_GET_DATALIST
    } from '../config/api'

    const LANGS = {
        zh: {
            addCity: '+ 增加城市',
            feedback: '加群反馈',
            title: '前端框架招聘热度',
            subtitle: '数据来源: 各大主流招聘平台',
            trendTitle: '职位趋势',
            cities: { sh: '上海', bj: '北京', sz: '深圳', gz: '广州' },
            all: '全部',
            loading: '数据加载中...',
            dataUpdateHint: '数据每天凌晨更新',
            yesterday: '昨天',
            rankTitle: '城市职位排行',
            cityCol: '城市',
            jobCount: '职位数',
            reactTrend: 'react趋势',
            vueTrend: 'vue趋势',
            citiesRank: {
                cq: '重庆', gzgz: '赣州', hz: '杭州', cd: '成都', tj: '天津',
                szsz: '苏州', nj: '南京', gy: '贵阳', cs: '长沙', sy: '沈阳',
                jn: '济南', nb: '宁波', wx: '无锡', cz: '郴州', dl: '大连',
                hf: '合肥', wh: '武汉'
            },
            expandTable: '展开排行',
            collapseTable: '收起排行',
            view3d: '3D 柱图',
            view3dGlobe: '3D 地球',
            back2d: '← 2D 视图',
            hint3d: '拖拽旋转 · 滚轮缩放',
        },
        en: {
            addCity: '+ Add City',
            feedback: 'Feedback',
            title: 'Frontend Framework Job Trends',
            subtitle: 'Data source: Major recruitment platforms',
            trendTitle: 'Job Trends',
            cities: { sh: 'Shanghai', bj: 'Beijing', sz: 'Shenzhen', gz: 'Guangzhou' },
            all: 'All',
            loading: 'Loading data...',
            dataUpdateHint: 'Data updates daily at midnight',
            yesterday: 'Yesterday',
            rankTitle: 'National City Job Rankings',
            cityCol: 'City',
            jobCount: 'Jobs',
            reactTrend: 'React Trend',
            vueTrend: 'Vue Trend',
            citiesRank: {
                cq: 'Chongqing', gzgz: 'Ganzhou', hz: 'Hangzhou', cd: 'Chengdu', tj: 'Tianjin',
                szsz: 'Suzhou', nj: 'Nanjing', gy: 'Guiyang', cs: 'Changsha', sy: 'Shenyang',
                jn: "Ji'nan", nb: 'Ningbo', wx: 'Wuxi', cz: 'Chenzhou', dl: 'Dalian',
                hf: 'Hefei', wh: 'Wuhan'
            },
            expandTable: 'Expand',
            collapseTable: 'Collapse',
            view3d: '3D Bars',
            view3dGlobe: '3D Globe',
            back2d: '← 2D View',
            hint3d: 'Drag to rotate · Scroll to zoom',
        },
        fr: {
            addCity: '+ Ajouter ville',
            feedback: 'Commentaires',
            title: 'Tendances emplois frontend',
            subtitle: 'Source: Principales plateformes de recrutement',
            trendTitle: 'Tendances des postes',
            cities: { sh: 'Shanghai', bj: 'Pékin', sz: 'Shenzhen', gz: 'Guangzhou' },
            all: 'Tout',
            loading: 'Chargement...',
            dataUpdateHint: 'Données mises à jour chaque nuit',
            yesterday: 'Hier',
            rankTitle: 'Classement national des villes',
            cityCol: 'Ville',
            jobCount: 'Postes',
            reactTrend: 'Tendance React',
            vueTrend: 'Tendance Vue',
            citiesRank: {
                cq: 'Chongqing', gzgz: 'Ganzhou', hz: 'Hangzhou', cd: 'Chengdu', tj: 'Tianjin',
                szsz: 'Suzhou', nj: 'Nanjing', gy: 'Guiyang', cs: 'Changsha', sy: 'Shenyang',
                jn: 'Jinan', nb: 'Ningbo', wx: 'Wuxi', cz: 'Chenzhou', dl: 'Dalian',
                hf: 'Hefei', wh: 'Wuhan'
            },
            expandTable: 'Étendre',
            collapseTable: 'Réduire',
            view3d: '3D Barres',
            view3dGlobe: '3D Globe',
            back2d: '← Vue 2D',
            hint3d: 'Glisser pour tourner · Molette pour zoomer',
        },
        ko: {
            addCity: '+ 도시 추가',
            feedback: '피드백',
            title: '프론트엔드 프레임워크 채용 동향',
            subtitle: '데이터 출처: 주요 채용 플랫폼',
            trendTitle: '채용 동향',
            cities: { sh: '상하이', bj: '베이징', sz: '선전', gz: '광저우' },
            all: '전체',
            loading: '데이터 로딩 중...',
            dataUpdateHint: '데이터는 매일 자정에 업데이트됩니다',
            yesterday: '어제',
            rankTitle: '전국 도시 채용 순위',
            cityCol: '도시',
            jobCount: '채용수',
            reactTrend: 'React 추세',
            vueTrend: 'Vue 추세',
            citiesRank: {
                cq: '충칭', gzgz: '간저우', hz: '항저우', cd: '청두', tj: '톈진',
                szsz: '쑤저우', nj: '난징', gy: '구이양', cs: '창사', sy: '선양',
                jn: '지난', nb: '닝보', wx: '우시', cz: '천저우', dl: '다롄',
                hf: '허페이', wh: '우한'
            },
            expandTable: '펼치기',
            collapseTable: '접기',
            view3d: '3D 막대',
            view3dGlobe: '3D 지구본',
            back2d: '← 2D 뷰',
            hint3d: '드래그하여 회전 · 스크롤하여 확대',
        },
        nl: {
            addCity: '+ Stad toevoegen',
            feedback: 'Feedback',
            title: 'Frontend Framework Vacaturetrends',
            subtitle: 'Gegevensbron: Grote wervingsplatforms',
            trendTitle: 'Vacaturetrends',
            cities: { sh: 'Shanghai', bj: 'Peking', sz: 'Shenzhen', gz: 'Guangzhou' },
            all: 'Alles',
            loading: 'Laden...',
            dataUpdateHint: 'Gegevens worden elke nacht bijgewerkt',
            yesterday: 'Gisteren',
            rankTitle: 'Nationale stadsrangschikking',
            cityCol: 'Stad',
            jobCount: 'Vacatures',
            reactTrend: 'React trend',
            vueTrend: 'Vue trend',
            citiesRank: {
                cq: 'Chongqing', gzgz: 'Ganzhou', hz: 'Hangzhou', cd: 'Chengdu', tj: 'Tianjin',
                szsz: 'Suzhou', nj: 'Nanjing', gy: 'Guiyang', cs: 'Changsha', sy: 'Shenyang',
                jn: 'Jinan', nb: 'Ningbo', wx: 'Wuxi', cz: 'Chenzhou', dl: 'Dalian',
                hf: 'Hefei', wh: 'Wuhan'
            },
            expandTable: 'Uitvouwen',
            collapseTable: 'Inklappen',
            view3d: '3D Staafdiagram',
            view3dGlobe: '3D Wereldbol',
            back2d: '← 2D Weergave',
            hint3d: 'Slepen om te draaien · Scrollen om te zoomen',
        },
        no: {
            addCity: '+ Legg til by',
            feedback: 'Tilbakemelding',
            title: 'Frontend-rammeverk jobbtrender',
            subtitle: 'Datakilde: Store rekrutteringsplattformer',
            trendTitle: 'Jobbtrender',
            cities: { sh: 'Shanghai', bj: 'Beijing', sz: 'Shenzhen', gz: 'Guangzhou' },
            all: 'Alle',
            loading: 'Laster...',
            dataUpdateHint: 'Data oppdateres hver natt',
            yesterday: 'I går',
            rankTitle: 'Nasjonal byrangering',
            cityCol: 'By',
            jobCount: 'Jobber',
            reactTrend: 'React trend',
            vueTrend: 'Vue trend',
            citiesRank: {
                cq: 'Chongqing', gzgz: 'Ganzhou', hz: 'Hangzhou', cd: 'Chengdu', tj: 'Tianjin',
                szsz: 'Suzhou', nj: 'Nanjing', gy: 'Guiyang', cs: 'Changsha', sy: 'Shenyang',
                jn: 'Jinan', nb: 'Ningbo', wx: 'Wuxi', cz: 'Chenzhou', dl: 'Dalian',
                hf: 'Hefei', wh: 'Wuhan'
            },
            expandTable: 'Utvid',
            collapseTable: 'Skjul',
            view3d: '3D Søyler',
            view3dGlobe: '3D Globus',
            back2d: '← 2D Visning',
            hint3d: 'Dra for å rotere · Rull for å zoome',
        },
    }

    export default {
        name: "index",
        components:{
            NumberGrow,
            NumberGrows,
            Chart3D,
            Globe3D,
    },
        data () {
            return {
                loading: true,
                show: true,
                count: "",
                TIME_COUNT:600,
                timer: null,
                data:'',
                datas:'',
                data_r:'',
                data_rs:'',
                dataArrvue:[],
                dataArrreact:[],
                timeDtae:[],
                vueData:[],
                reactData:[],
                sh_vue_data:[],
                sh_react_data:[],
                bj_vue_data:[],
                bj_react_data:[],
                sz_vue_data:[],
                sz_react_data:[],
                gz_vue_data:[],
                gz_react_data:[],
                sh_old:'',
                bj_old:'',
                sz_old:'',
                gz_old:'',
                sh_oldval:'',
                bj_oldval:'',
                sz_oldval:'',
                gz_oldval:'',
                sh_old_v:'',
                bj_old_v:'',
                sz_old_v:'',
                gz_old_v:'',
                data_rs_more:'',
                moreHistory: [],
                selectedCity: 'bj',
                sh_oldval_v:'',
                bj_oldval_v:'',
                sz_oldval_v:'',
                gz_oldval_v:'',
                rotate: 90,
                align: 'left',
                verticalAlign: 'middle',
                position: 'insideBottom',
                distance: 15,
                obgdata:{},
                charts: '',
                isac:false,
                selectedYear: 'all',
                tableCollapsed: false,
                view3DMode: null,
                currentLang: 'zh',
                langMenuOpen: false,
                langLabels: {
                    zh: '中文',
                    en: 'English',
                    fr: 'Français',
                    ko: '한국어',
                    nl: 'Nederlands',
                    no: 'Norsk',
                },
            }
        },
        computed:{
            t() {
                return LANGS[this.currentLang]
            },
            selectedMoreCity() {
                return Object.prototype.hasOwnProperty.call(this.t.citiesRank, this.selectedCity) ? this.selectedCity : '';
            },
            availableYears() {
                const years = [...new Set(this.timeDtae.map(d => d ? d.substring(0, 4) : ''))].filter(Boolean);
                return years.sort();
            },
            filteredIndices() {
                if (this.selectedYear === 'all' || !this.timeDtae.length) {
                    return this.timeDtae.map((_, i) => i);
                }
                return this.timeDtae.reduce((acc, d, i) => {
                    if (d && d.startsWith(this.selectedYear)) acc.push(i);
                    return acc;
                }, []);
            },
            filteredTimeDtae() {
                return this.filteredIndices.map(i => this.timeDtae[i]);
            },
            filteredVueData() {
                return this.filteredIndices.map(i => this.vueData[i]);
            },
            filteredReactData() {
                return this.filteredIndices.map(i => this.reactData[i]);
            },
            allCityData3D() {
                if (!this.data || !this.data_rs_more) return []
                const last = this.data[this.data.length - 1]
                const r = JSON.parse(last.react)
                const v = JSON.parse(last.vue)
                const mr = JSON.parse(this.data_rs_more.react)
                const mv = JSON.parse(this.data_rs_more.vue)
                const cr = this.t.citiesRank
                return [
                    { name: this.t.cities.sh,  react: r.sh,       vue: v.sh,       lat: 31.2,  lon: 121.5 },
                    { name: this.t.cities.bj,  react: r.bj,       vue: v.bj,       lat: 39.9,  lon: 116.4 },
                    { name: this.t.cities.sz,  react: r.sz,        vue: v.sz,       lat: 22.5,  lon: 114.1 },
                    { name: this.t.cities.gz,  react: r.gz,        vue: v.gz,       lat: 23.1,  lon: 113.3 },
                    { name: cr.cq,             react: mr.cq  ||0,  vue: mv.cq  ||0, lat: 29.6,  lon: 106.5 },
                    { name: cr.hz,             react: mr.hz  ||0,  vue: mv.hz  ||0, lat: 30.3,  lon: 120.2 },
                    { name: cr.cd,             react: mr.cd  ||0,  vue: mv.cd  ||0, lat: 30.6,  lon: 104.1 },
                    { name: cr.tj,             react: mr.tj  ||0,  vue: mv.tj  ||0, lat: 39.1,  lon: 117.2 },
                    { name: cr.szsz,           react: mr.szsz||0,  vue: mv.szsz||0, lat: 31.3,  lon: 120.6 },
                    { name: cr.nj,             react: mr.nj  ||0,  vue: mv.nj  ||0, lat: 32.1,  lon: 118.8 },
                    { name: cr.wh,             react: mr.wh  ||0,  vue: mv.wh  ||0, lat: 30.6,  lon: 114.3 },
                    { name: cr.hf,             react: mr.hf  ||0,  vue: mv.hf  ||0, lat: 31.9,  lon: 117.3 },
                    { name: cr.cs,             react: mr.cs  ||0,  vue: mv.cs  ||0, lat: 28.2,  lon: 113.0 },
                    { name: cr.jn,             react: mr.jn  ||0,  vue: mv.jn  ||0, lat: 36.7,  lon: 117.1 },
                    { name: cr.nb,             react: mr.nb  ||0,  vue: mv.nb  ||0, lat: 29.9,  lon: 121.5 },
                    { name: cr.wx,             react: mr.wx  ||0,  vue: mv.wx  ||0, lat: 31.6,  lon: 120.3 },
                    { name: cr.dl,             react: mr.dl  ||0,  vue: mv.dl  ||0, lat: 38.9,  lon: 121.6 },
                    { name: cr.sy,             react: mr.sy  ||0,  vue: mv.sy  ||0, lat: 41.8,  lon: 123.4 },
                    { name: cr.gy,             react: mr.gy  ||0,  vue: mv.gy  ||0, lat: 26.6,  lon: 106.7 },
                    { name: cr.gzgz,           react: mr.gzgz||0,  vue: mv.gzgz||0, lat: 25.8,  lon: 115.0 },
                    { name: cr.cz,             react: mr.cz  ||0,  vue: mv.cz  ||0, lat: 25.8,  lon: 113.0 },
                ]
            },
        },
        watch: {
            timeDtae() {
                this.$nextTick(function () {
                    this.drawPie('main')
                })
            },
            selectedYear() {
                this.$nextTick(function () {
                    this.drawPie('main')
                })
            },
            currentLang() {
                this.$nextTick(function () {
                    this.drawPie('main')
                })
            },
            sh_old: {
                handler: function (val, oldval) {
                    this.sh_oldval = oldval ? oldval : '--'
                },
            },
            bj_old: {
                handler: function (val, oldval) {
                    this.bj_oldval = oldval ? oldval : '--'
                },
            },
            sz_old: {
                handler: function (val, oldval) {
                    this.sz_oldval = oldval ? oldval : '--'
                },
            },
            gz_old: {
                handler: function (val, oldval) {
                    this.gz_oldval = oldval ? oldval : '--'
                },
            },
            sh_old_v: {
                handler: function (val, oldval) {
                    this.sh_oldval_v = oldval ? oldval : '--'
                },
            },
            bj_old_v: {
                handler: function (val, oldval) {
                    this.bj_oldval_v = oldval ? oldval : '--'
                },
            },
            sz_old_v: {
                handler: function (val, oldval) {
                    this.sz_oldval_v = oldval ? oldval : '--'
                },
            },
            gz_old_v: {
                handler: function (val, oldval) {
                    this.gz_oldval_v = oldval ? oldval : '--'
                },
            },
        },
        created(){
            this.currentLang = this.detectLang()
        },
        mounted() {
            this.getCode()
            this.loadAllData()

            this.$nextTick(function() {
                this.drawPie('main')
            })

            window.addEventListener("resize",()=>{
                if(this.charts) this.charts.resize();
            })

        },

            methods:{
                detectLang() {
                    const lang = (navigator.language || navigator.userLanguage || 'zh').toLowerCase()
                    const code = lang.split('-')[0]
                    if (code === 'zh') return 'zh'
                    if (code === 'fr') return 'fr'
                    if (code === 'ko') return 'ko'
                    if (code === 'nl') return 'nl'
                    if (code === 'no' || code === 'nb' || code === 'nn') return 'no'
                    if (code === 'en') return 'en'
                    return 'en'
                },
                setLang(code) {
                    this.currentLang = code
                    this.langMenuOpen = false
                },
                handleGlobalClick() {
                    if (this.langMenuOpen) this.langMenuOpen = false
                },
                addToList() {
                    this.toadd();
                },
                drawPie(id){
                    var zoomStart = 0;
                    var zoomEnd = 100;
                    if(this.charts){
                        var option = this.charts.getOption();
                        if(option && option.dataZoom && option.dataZoom.length > 0){
                            zoomStart = option.dataZoom[0].start;
                            zoomEnd = option.dataZoom[0].end;
                        }
                        this.charts.dispose();
                    }
                    this.charts = echarts.init(document.getElementById(id))
                    var isMobile = window.innerWidth <= 767;
                    var isSmall = window.innerWidth <= 480;
                    var labelFontSize = isSmall ? 10 : (isMobile ? 11 : 13);
                    var showLabel = !isSmall;
                    const filteredTime = this.filteredTimeDtae;
                    const filteredVue = this.filteredVueData;
                    const filteredReact = this.filteredReactData;
                    const vueTrend = this.linearRegression(filteredVue);
                    const reactTrend = this.linearRegression(filteredReact);
                    const reactTrendName = this.t.reactTrend;
                    const vueTrendName = this.t.vueTrend;
                    this.charts.setOption({
                            grid:{
                                x: isSmall ? 40 : (isMobile ? 50 : 65),
                                y: isMobile ? 45 : 55,
                                x2: isSmall ? 15 : (isMobile ? 25 : 40),
                                y2: isMobile ? 35 : 40,
                            },
                        tooltip: {
                            trigger: 'axis',
                            backgroundColor: 'rgba(0,15,40,0.85)',
                            borderColor: 'rgba(0,255,255,0.3)',
                            borderWidth: 1,
                            textStyle: {
                                color: '#fff'
                            },
                            axisPointer: {
                                type: 'cross',
                                crossStyle: {
                                    color: '#00fff2'
                                },
                                lineStyle: {
                                    color: 'rgba(0,255,242,0.3)'
                                }
                            },
                            formatter: (params) => {
                                let result = params[0].axisValueLabel + '<br/>';
                                params.forEach(item => {
                                    if (item.seriesName === reactTrendName || item.seriesName === vueTrendName) return;
                                    const val = item.value;
                                    result += item.marker + item.seriesName + ': ' + val + '<br/>';
                                });
                                return result;
                            }
                        },
                        toolbox: {
                            itemSize:16,
                            feature: {
                                magicType: {
                                    show: true, type: ['line', 'bar'],
                                    seriesIndex: { line: [0, 1], bar: [0, 1] },
                                    iconStyle:{
                                        color:'#fff',
                                        borderColor:'rgba(255,255,255,0.5)'
                                    },
                                    emphasis:{
                                        iconStyle:{
                                            color:'#00fff2',
                                            borderColor:'#00fff2'
                                        }
                                    }
                                }
                            },
                            top:10,
                            left:130
                        },
                        legend: {
                            padding: isMobile ? [10,0,0,0] : [15,0,0,0],
                            data: ['react', 'vue'],
                            textStyle:{
                                fontSize: isMobile ? '12' : '16',
                                color: 'rgba(255,255,255,0.85)'
                            },
                            itemWidth: isMobile ? 16 : 25,
                            itemHeight: isMobile ? 10 : 14,
                        },
                        xAxis: [
                            {
                                type: 'category',
                                data: filteredTime,
                                axisPointer: {
                                    type: 'shadow'
                                },
                                axisLabel: {
                                    show: true,
                                    textStyle: {
                                        color: 'rgba(255,255,255,0.6)',
                                        fontSize: isMobile ? 10 : 12
                                    },
                                    rotate: isMobile ? 30 : 0
                                },
                                axisLine: {
                                    lineStyle: {
                                        color: 'rgba(255,255,255,0.15)'
                                    }
                                },
                                splitLine: {
                                    lineStyle: {
                                        color: 'rgba(255,255,255,0.05)'
                                    }
                                }
                            }
                        ],
                        yAxis: [
                            {
                                type: 'value',
                                name: isMobile ? '' : this.t.jobCount,
                                nameTextStyle: {
                                    color: 'rgba(255,255,255,0.6)'
                                },
                                min: 0,
                                minInterval: 1,
                                axisLabel: {
                                    formatter: isSmall ? '{value}' : ('{value}'),
                                    show: true,
                                    textStyle: {
                                        color: 'rgba(255,255,255,0.6)',
                                        fontSize: isMobile ? 10 : 12
                                    }
                                },
                                axisLine: {
                                    lineStyle: {
                                        color: 'rgba(255,255,255,0.15)'
                                    }
                                },
                                splitLine: {
                                    lineStyle: {
                                        color: 'rgba(255,255,255,0.06)'
                                    }
                                }
                            },
                        ],
                        dataZoom: [
                            {
                                type: 'inside',
                                start: zoomStart,
                                end: zoomEnd
                            },
                            {
                                show: !isSmall,
                                textStyle:{
                                    color:'#00fff2',
                                },
                                type: 'slider',
                                top: '90%',
                                start: zoomStart,
                                end: zoomEnd,
                                borderColor: 'rgba(0,255,242,0.2)',
                                fillerColor: 'rgba(0,255,242,0.08)',
                                handleStyle: {
                                    color: '#00fff2'
                                }
                            }
                        ],
                        series: [
                            {
                                name: 'react',
                                type: 'line',
                                smooth: true,
                                data: filteredReact,
                                lineStyle: {
                                    normal: {
                                        color: '#ff4757',
                                        width: 2,
                                        shadowColor: 'rgba(255,71,87,0.5)',
                                        shadowBlur: isMobile ? 4 : 10,
                                    },
                                },
                                areaStyle: {
                                    normal: {
                                        color: new echarts.graphic.LinearGradient(0, 0, 0, 1, [
                                            { offset: 0, color: 'rgba(255,71,87,0.25)' },
                                            { offset: 1, color: 'rgba(255,71,87,0)' }
                                        ])
                                    }
                                },
                                itemStyle: {
                                    normal: {
                                        color: '#ff4757',
                                        label: {
                                            show: showLabel,
                                            position: 'top',
                                            formatter: (params) => params.value,
                                            textStyle: {
                                                color: '#ff6b7a',
                                                fontSize: labelFontSize
                                                }
                                            }
                                          }
                                    },
                            },
                            {
                                name: 'vue',
                                type: 'line',
                                smooth: true,
                                data: filteredVue,
                                lineStyle: {
                                    normal: {
                                        color: '#ffd32a',
                                        width: 2,
                                        shadowColor: 'rgba(255,211,42,0.5)',
                                        shadowBlur: isMobile ? 4 : 10,
                                    },
                                },
                                areaStyle: {
                                    normal: {
                                        color: new echarts.graphic.LinearGradient(0, 0, 0, 1, [
                                            { offset: 0, color: 'rgba(255,211,42,0.2)' },
                                            { offset: 1, color: 'rgba(255,211,42,0)' }
                                        ])
                                    }
                                },
                                itemStyle: {
                                    normal: {
                                        color: '#ffd32a',
                                        label: {
                                            show: showLabel,
                                            position: 'top',
                                            formatter: (params) => params.value,
                                            textStyle: {
                                                color: '#ffd32a',
                                                fontSize: labelFontSize
                                            }
                                        }
                                    }
                                },
                            },
                            {
                                name: reactTrendName,
                                type: 'line',
                                data: reactTrend,
                                smooth: false,
                                symbol: 'none',
                                legendHoverLink: false,
                                itemStyle: { normal: { color: '#ff4757' } },
                                lineStyle: {
                                    normal: {
                                        color: '#ff4757',
                                        width: 2,
                                        type: 'dashed',
                                        opacity: 0.55,
                                    }
                                },
                            },
                            {
                                name: vueTrendName,
                                type: 'line',
                                data: vueTrend,
                                smooth: false,
                                symbol: 'none',
                                legendHoverLink: false,
                                itemStyle: { normal: { color: '#ffd32a' } },
                                lineStyle: {
                                    normal: {
                                        color: '#ffd32a',
                                        width: 2,
                                        type: 'dashed',
                                        opacity: 0.55,
                                    }
                                },
                            },
                        ]
                    })
                },
            loadAllData(){
                this.loading = true;
                Promise.all([
                    this.getlistDates_more(),
                    this.getListDates()
                ]).then(() => {
                    this.loading = false;
                }).catch(() => {
                    this.loading = false;
                });
            },
            getlistDates_more(){
                return this.$get(P_GET_DATALIST_MORE, { history: 1 }).then(res => {
                    if (res.code != 200 || !Array.isArray(res.data)) throw new Error('Invalid city history');
                    this.moreHistory = res.data.slice().sort((a, b) => a.ceate_time.localeCompare(b.ceate_time));
                    this.data_rs_more = this.moreHistory[this.moreHistory.length - 1] || '';
                    this.selectCity(this.selectedCity);
                });
            },
            getListDates(){
                return this.$get(P_GET_DATALIST).then(res => {
                    if (res.code != 200 || !Array.isArray(res.data)) throw new Error('Invalid city history');
                    this.data = res.data.length ? res.data : '';
                    this.selectCity(this.selectedCity);
                });
            },
            selectCity(city){
                if (!Object.prototype.hasOwnProperty.call(this.t.cities, city) &&
                    !Object.prototype.hasOwnProperty.call(this.t.citiesRank, city)) return;
                this.selectedCity = city;
                const rows = (this.selectedMoreCity ? this.moreHistory : this.data) || [];
                const value = (row, framework) => {
                    const values = typeof row[framework] === 'string' ? JSON.parse(row[framework]) : row[framework];
                    const raw = values && values[city];
                    if (raw === undefined || raw === null || raw === '') return null;
                    const number = Number(String(raw).replace(/,/g, '').replace(/\+$/, ''));
                    return Number.isFinite(number) ? number : null;
                };
                this.timeDtae = rows.map(row => row.ceate_time);
                this.vueData = rows.map(row => value(row, 'vue'));
                this.reactData = rows.map(row => value(row, 'react'));
                if (this.selectedYear !== 'all' && !this.availableYears.includes(this.selectedYear)) this.selectedYear = 'all';
            },

                getCode() {
                    if (!this.timer) {
                        this.count = this.TIME_COUNT;
                        this.show = false;
                        this.timer = setInterval(() => {
                            if (this.count > 0 && this.count <= this.TIME_COUNT) {
                                this.count -= 1;
                            } else {
                                this.show = true;
                                clearInterval(this.timer);
                                this.timer = null;
                                this.getCode()
                            }
                        }, 1000);
                    }
                },
            toadd(){
                if(this.isac){
                    this.isac = false
                }else {
                    this.isac = true;
                }
            },
            strs(str){
                let numArr = str ? str.match(/\d+/g) : '';
                return numArr ?  +numArr.join('') : '0'
            },
            linearRegression(data) {
                const valid = data.map((value, i) => ({ value, i })).filter(point => point.value !== null && Number.isFinite(Number(point.value)));
                if (valid.length < 2) return [];
                const first = valid[0];
                const last = valid[valid.length - 1];
                return data.map((_, i) => i < first.i || i > last.i ? null : Math.round(Number(first.value) + (Number(last.value) - Number(first.value)) * (i - first.i) / (last.i - first.i)));
            },
            setYear(year) {
                this.selectedYear = year;
            },
             tasterNumber(str){
                 return str.replace(/[^0-9]/ig,"");
             },
            toggleTable() {
                this.tableCollapsed = !this.tableCollapsed;
                setTimeout(() => {
                    if (this.charts) this.charts.resize();
                }, 370);
            },

        }
    }
</script>

<style scoped>
/* ============================================
   DASHBOARD - 赛博朋克仪表盘主题
   ============================================ */

/* 扫描线动效 */
.scanline {
    position: fixed;
    top: 0;
    left: 0;
    right: 0;
    height: 3px;
    background: linear-gradient(90deg, transparent, rgba(0, 255, 242, 0.4), transparent);
    animation: scanMove 4s linear infinite;
    z-index: 9999;
    pointer-events: none;
}
@keyframes scanMove {
    0% { transform: translateY(-100%); opacity: 0; }
    10% { opacity: 1; }
    90% { opacity: 1; }
    100% { transform: translateY(100vh); opacity: 0; }
}

/* 全局仪表盘容器 */
.dashboard {
    width: 100%;
    min-height: 100vh;
    background: #0a0e27;
    background-image:
        radial-gradient(ellipse at 10% 20%, rgba(0, 100, 255, 0.08) 0%, transparent 50%),
        radial-gradient(ellipse at 90% 80%, rgba(0, 255, 242, 0.05) 0%, transparent 50%),
        radial-gradient(ellipse at 50% 50%, rgba(20, 20, 60, 0.5) 0%, transparent 70%);
    padding: 16px;
    box-sizing: border-box;
    display: flex;
    flex-direction: column;
    gap: 16px;
    overflow-x: hidden;
}

/* 玻璃质感卡片 */
.glass-card {
    background: rgba(10, 20, 50, 0.65);
    backdrop-filter: blur(12px);
    -webkit-backdrop-filter: blur(12px);
    border: 1px solid rgba(0, 255, 242, 0.12);
    border-radius: 12px;
    box-shadow:
        0 0 20px rgba(0, 0, 0, 0.3),
        inset 0 1px 0 rgba(255, 255, 255, 0.05);
    transition: border-color 0.3s ease, box-shadow 0.3s ease;
}
.glass-card:hover {
    border-color: rgba(0, 255, 242, 0.25);
    box-shadow:
        0 0 30px rgba(0, 255, 242, 0.08),
        0 0 20px rgba(0, 0, 0, 0.3),
        inset 0 1px 0 rgba(255, 255, 255, 0.05);
}

/* ============ 头部 ============ */
.dash-header {
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: 12px 24px;
    background: rgba(10, 20, 50, 0.5);
    border: 1px solid rgba(0, 255, 242, 0.08);
    border-radius: 12px;
    min-height: 60px;
    flex-shrink: 0;
}

.header-left {
    display: flex;
    align-items: center;
    gap: 12px;
    position: relative;
}

.neon-btn {
    position: relative;
    padding: 8px 22px;
    font-size: 13px;
    font-weight: 600;
    color: #00fff2;
    background: transparent;
    border: 1px solid rgba(0, 255, 242, 0.4);
    border-radius: 6px;
    cursor: pointer;
    overflow: hidden;
    transition: all 0.3s ease;
    letter-spacing: 1px;
    font-family: inherit;
}
.neon-btn:hover {
    background: rgba(0, 255, 242, 0.1);
    border-color: #00fff2;
    box-shadow: 0 0 20px rgba(0, 255, 242, 0.2), inset 0 0 20px rgba(0, 255, 242, 0.05);
}
.neon-btn .btn-glow {
    position: absolute;
    top: -50%;
    left: -50%;
    width: 200%;
    height: 200%;
    background: conic-gradient(transparent, rgba(0, 255, 242, 0.1), transparent 30%);
    animation: btnRotate 4s linear infinite;
}
@keyframes btnRotate {
    100% { transform: rotate(360deg); }
}
.btn-text {
    position: relative;
    z-index: 1;
}

.feedback-link {
    opacity: 0;
    transform: translateX(-10px);
    transition: all 0.3s ease;
    pointer-events: none;
}
.feedback-link.show {
    opacity: 1;
    transform: translateX(0);
    pointer-events: auto;
}
.feedback-link a {
    color: rgba(0, 255, 242, 0.7);
    text-decoration: none;
    font-size: 12px;
    border: 1px solid rgba(0, 255, 242, 0.2);
    padding: 5px 14px;
    border-radius: 4px;
    transition: all 0.2s;
}
.feedback-link a:hover {
    color: #00fff2;
    border-color: #00fff2;
}

.header-center {
    text-align: center;
    flex: 1;
}
.title-wrapper {
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 20px;
}
.title-decoration {
    width: 80px;
    height: 2px;
    background: linear-gradient(90deg, transparent, rgba(0, 255, 242, 0.6));
    position: relative;
}
.title-decoration.right {
    background: linear-gradient(90deg, rgba(0, 255, 242, 0.6), transparent);
}
.title-decoration::after {
    content: '';
    position: absolute;
    top: -2px;
    width: 6px;
    height: 6px;
    border-radius: 50%;
    background: #00fff2;
    box-shadow: 0 0 10px #00fff2;
}
.title-decoration.left::after {
    right: 0;
}
.title-decoration.right::after {
    left: 0;
}

.dash-title {
    font-size: 24px;
    font-weight: 700;
    letter-spacing: 6px;
    color: #fff;
    text-shadow: 0 0 30px rgba(0, 255, 242, 0.3);
    margin: 0;
    white-space: nowrap;
}
.dash-subtitle {
    font-size: 11px;
    color: rgba(255, 175, 0, 0.4);
    margin-top: 4px;
    letter-spacing: 2px;
}

.header-right {
    display: flex;
    align-items: center;
    gap: 12px;
}
.github-badges {
    display: flex;
    gap: 8px;
    opacity: 0.8;
}
.github-badges a {
    display: block;
    transition: opacity 0.2s;
}
.github-badges a:hover {
    opacity: 1;
}

/* ============ 语言切换器 ============ */
.lang-switcher {
    position: relative;
}
.lang-btn {
    display: flex;
    align-items: center;
    gap: 5px;
    padding: 6px 12px;
    font-size: 12px;
    font-weight: 600;
    color: rgba(0, 255, 242, 0.8);
    background: transparent;
    border: 1px solid rgba(0, 255, 242, 0.25);
    border-radius: 6px;
    cursor: pointer;
    transition: all 0.25s ease;
    font-family: inherit;
    white-space: nowrap;
}
.lang-btn:hover {
    color: #00fff2;
    border-color: rgba(0, 255, 242, 0.6);
    background: rgba(0, 255, 242, 0.06);
}
.lang-icon {
    font-size: 14px;
    line-height: 1;
}
.lang-arrow {
    font-size: 10px;
    transition: transform 0.2s ease;
    display: inline-block;
}
.lang-arrow.open {
    transform: rotate(180deg);
}
.lang-dropdown {
    position: absolute;
    top: calc(100% + 6px);
    right: 0;
    background: rgba(8, 16, 45, 0.97);
    border: 1px solid rgba(0, 255, 242, 0.25);
    border-radius: 8px;
    overflow: hidden;
    z-index: 1000;
    min-width: 130px;
    box-shadow: 0 8px 32px rgba(0, 0, 0, 0.5), 0 0 20px rgba(0, 255, 242, 0.06);
}
.lang-dropdown button {
    display: block;
    width: 100%;
    padding: 9px 16px;
    font-size: 13px;
    color: rgba(255, 255, 255, 0.65);
    background: transparent;
    border: none;
    border-bottom: 1px solid rgba(0, 255, 242, 0.06);
    cursor: pointer;
    text-align: left;
    transition: all 0.2s ease;
    font-family: inherit;
}
.lang-dropdown button:last-child {
    border-bottom: none;
}
.lang-dropdown button:hover {
    background: rgba(0, 255, 242, 0.08);
    color: #00fff2;
}
.lang-dropdown button.active {
    color: #00fff2;
    background: rgba(0, 255, 242, 0.1);
    font-weight: 600;
}

/* ============ 主体 ============ */
.dash-body {
    display: flex;
    gap: 16px;
    flex: 1;
    min-height: 0;
}

.dash-left {
    flex: 1;
    display: flex;
    flex-direction: column;
    gap: 16px;
    min-width: 0;
}

.dash-right {
    width: 320px;
    flex-shrink: 0;
    transition: width 0.35s cubic-bezier(0.4, 0, 0.2, 1);
    overflow: hidden;
}
.dash-right.collapsed {
    width: 44px;
}
.dash-right.collapsed .panel-title {
    opacity: 0;
    width: 0;
    overflow: hidden;
    white-space: nowrap;
    transition: opacity 0.2s ease, width 0.35s ease;
}
.dash-right.collapsed .panel-header {
    padding: 14px 8px;
    justify-content: center;
}
.dash-right.collapsed .table-scroll {
    opacity: 0;
    pointer-events: none;
}
.table-toggle-btn {
    flex-shrink: 0;
    width: 28px;
    height: 28px;
    display: flex;
    align-items: center;
    justify-content: center;
    background: rgba(0, 255, 242, 0.08);
    border: 1px solid rgba(0, 255, 242, 0.3);
    border-radius: 4px;
    color: #00fff2;
    cursor: pointer;
    font-size: 11px;
    transition: background 0.2s ease, border-color 0.2s ease;
    font-family: inherit;
}
.table-toggle-btn:hover {
    background: rgba(0, 255, 242, 0.18);
    border-color: #00fff2;
    box-shadow: 0 0 10px rgba(0, 255, 242, 0.2);
}

/* ============ 面板头部 ============ */
.panel-header {
    display: flex;
    align-items: center;
    justify-content: space-between;
    flex-wrap: wrap;
    gap: 10px;
    padding: 14px 20px;
    border-bottom: 1px solid rgba(0, 255, 242, 0.08);
}
.panel-title {
    font-size: 15px;
    font-weight: 600;
    color: rgba(255, 255, 255, 0.9);
    display: flex;
    align-items: center;
    gap: 10px;
    letter-spacing: 1px;
    transition: opacity 0.2s ease, width 0.35s ease;
    overflow: hidden;
    white-space: nowrap;
}
.dot {
    width: 8px;
    height: 8px;
    border-radius: 50%;
    background: #00fff2;
    box-shadow: 0 0 10px #00fff2;
}
.dot.cyan {
    background: #00fff2;
}
.dot.pulse {
    animation: dotPulse 2s ease-in-out infinite;
}
@keyframes dotPulse {
    0%, 100% { opacity: 1; box-shadow: 0 0 10px #00fff2; }
    50% { opacity: 0.4; box-shadow: 0 0 4px #00fff2; }
}

/* ============ 3D 视图按钮 ============ */
.btn-3d {
    display: flex;
    align-items: center;
    gap: 5px;
    padding: 5px 14px;
    font-size: 12px;
    font-weight: 700;
    color: #ffaf00;
    background: transparent;
    border: 1px solid rgba(255, 175, 0, 0.35);
    border-radius: 5px;
    cursor: pointer;
    letter-spacing: 1px;
    font-family: inherit;
    transition: all 0.25s ease;
    flex-shrink: 0;
}
.btn-3d:hover:not(:disabled) {
    background: rgba(255, 175, 0, 0.1);
    border-color: #ffaf00;
    box-shadow: 0 0 14px rgba(255, 175, 0, 0.25);
}
.btn-3d:disabled {
    opacity: 0.35;
    cursor: default;
}
.btn-3d-cube {
    font-size: 14px;
    line-height: 1;
}
.btn-3d-group {
    display: flex;
    gap: 6px;
    flex-shrink: 0;
}
.btn-3d-globe {
    color: #00fff2;
    border-color: rgba(0, 255, 242, 0.35);
}
.btn-3d-globe:hover:not(:disabled) {
    background: rgba(0, 255, 242, 0.1);
    border-color: #00fff2;
    box-shadow: 0 0 14px rgba(0, 255, 242, 0.25);
}

/* ============ 年份筛选 ============ */
.year-tabs {
    display: flex;
    gap: 6px;
    padding: 6px 20px;
    border-bottom: 1px solid rgba(0, 255, 242, 0.06);
    flex-wrap: wrap;
}
.year-tabs button {
    padding: 3px 14px;
    font-size: 12px;
    color: rgba(255, 255, 255, 0.45);
    background: transparent;
    border: 1px solid rgba(255, 255, 255, 0.08);
    border-radius: 3px;
    cursor: pointer;
    transition: all 0.25s ease;
    font-family: inherit;
}
.year-tabs button:hover {
    color: rgba(255, 255, 255, 0.7);
    border-color: rgba(0, 255, 242, 0.25);
}
.year-tabs button.active {
    color: #00fff2;
    border-color: #00fff2;
    background: rgba(0, 255, 242, 0.08);
}

/* ============ 图表区域 ============ */
.chart-panel {
    flex: 1;
    display: flex;
    flex-direction: column;
    min-height: 380px;
}
.chart-container {
    flex: 1;
    position: relative;
    padding: 8px;
}

.city-tabs {
    display: flex;
    gap: 4px;
}
.city-select {
    max-width: 150px;
    min-height: 30px;
    flex-shrink: 0;
    padding: 5px 10px;
    color: #00fff2;
    background: #0a142e;
    border: 1px solid rgba(0, 255, 242, 0.4);
    border-radius: 4px;
    font: inherit;
    font-size: 13px;
    cursor: pointer;
}
.city-select:disabled { opacity: 0.5; cursor: wait; }
.city-select:focus-visible { outline: 2px solid #00fff2; outline-offset: 2px; }
.city-tabs button {
    padding: 5px 16px;
    font-size: 13px;
    color: rgba(255, 255, 255, 0.5);
    background: transparent;
    border: 1px solid rgba(255, 255, 255, 0.1);
    border-radius: 4px;
    cursor: pointer;
    transition: all 0.25s ease;
    font-family: inherit;
}
.city-tabs button:hover {
    color: rgba(255, 255, 255, 0.8);
    border-color: rgba(0, 255, 242, 0.3);
}
.city-tabs button.active {
    color: #00fff2;
    border-color: #00fff2;
    background: rgba(0, 255, 242, 0.1);
    box-shadow: 0 0 12px rgba(0, 255, 242, 0.15);
}

/* loading */
.chart-loading {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
    z-index: 10;
    background: rgba(10, 14, 39, 0.8);
    border-radius: 12px;
}
.chart-spinner {
    width: 36px;
    height: 36px;
    border: 3px solid rgba(0, 255, 242, 0.1);
    border-top-color: #00fff2;
    border-radius: 50%;
    animation: spin 0.8s linear infinite;
}
.chart-loading-text {
    color: rgba(255, 255, 255, 0.5);
    font-size: 13px;
    margin-top: 10px;
    letter-spacing: 2px;
}
@keyframes spin {
    to { transform: rotate(360deg); }
}

/* ============ 数据卡片区 ============ */
.city-cards {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(155px, 1fr));
    gap: 12px;
    flex-shrink: 0;
}

.stat-card {
    padding: 14px 16px;
    position: relative;
    overflow: hidden;
}

/* 卡片四角装饰 */
.card-corner {
    position: absolute;
    width: 12px;
    height: 12px;
    border-color: rgba(0, 255, 242, 0.4);
    border-style: solid;
    border-width: 0;
}
.card-corner.tl { top: 0; left: 0; border-top-width: 2px; border-left-width: 2px; }
.card-corner.tr { top: 0; right: 0; border-top-width: 2px; border-right-width: 2px; }
.card-corner.bl { bottom: 0; left: 0; border-bottom-width: 2px; border-left-width: 2px; }
.card-corner.br { bottom: 0; right: 0; border-bottom-width: 2px; border-right-width: 2px; }

.card-label {
    font-size: 14px;
    font-weight: 700;
    color: rgba(255, 255, 255, 0.9);
    letter-spacing: 2px;
    margin-bottom: 2px;
}
.card-hint {
    font-size: 10px;
    color: rgba(255, 255, 255, 0.3);
    margin-bottom: 6px;
}
.card-divider {
    height: 1px;
    background: linear-gradient(90deg, rgba(0, 255, 242, 0.3), transparent);
    margin: 8px 0;
}
.card-row {
    display: flex;
    align-items: center;
    justify-content: space-between;
    margin: 4px 0;
}

.framework-tag {
    font-size: 10px;
    font-weight: 700;
    padding: 2px 8px;
    border-radius: 3px;
    letter-spacing: 1px;
}
.framework-tag.react {
    color: #ff4757;
    background: rgba(255, 71, 87, 0.15);
    border: 1px solid rgba(255, 71, 87, 0.3);
}
.framework-tag.vue {
    color: #ffd32a;
    background: rgba(255, 211, 42, 0.12);
    border: 1px solid rgba(255, 211, 42, 0.3);
}

.card-value {
    font-size: 22px;
    font-weight: 700;
    color: #fff;
    font-family: 'Arial', monospace;
}
.card-value.dim {
    color: rgba(255, 255, 255, 0.2);
    font-size: 16px;
}
.card-value.react-val {
    color: #ff4757;
    text-shadow: 0 0 15px rgba(255, 71, 87, 0.4);
}
.card-value.vue-val {
    color: #ffd32a;
    text-shadow: 0 0 15px rgba(255, 211, 42, 0.4);
}
.plus-sign {
    font-size: 0.7em;
    opacity: 0.6;
    margin-left: 1px;
}

.card-yesterday {
    font-size: 11px;
    color: rgba(255, 255, 255, 0.3);
    text-align: right;
    margin-bottom: 4px;
}
.card-yesterday span {
    color: rgba(255, 225, 150, 0.6);
}

.info-card {
    background: rgba(10, 20, 50, 0.5);
    display: flex;
    flex-direction: column;
    justify-content: center;
}

/* ============ 右侧排行表 ============ */
.table-panel {
    height: 100%;
    display: flex;
    flex-direction: column;
}
.table-scroll {
    flex: 1;
    overflow-y: auto;
    padding: 0 4px;
    transition: opacity 0.2s ease;
}
.table-scroll::-webkit-scrollbar {
    width: 4px;
}
.table-scroll::-webkit-scrollbar-track {
    background: transparent;
}
.table-scroll::-webkit-scrollbar-thumb {
    background: rgba(0, 255, 242, 0.2);
    border-radius: 2px;
}

.rank-table {
    width: 100%;
    border-collapse: collapse;
}
.rank-table thead {
    position: sticky;
    top: 0;
    z-index: 2;
}
.rank-table th {
    padding: 10px 12px;
    font-size: 12px;
    font-weight: 600;
    color: rgba(255, 255, 255, 0.5);
    text-align: center;
    background: rgba(10, 14, 39, 0.95);
    border-bottom: 1px solid rgba(0, 255, 242, 0.15);
    letter-spacing: 1px;
    text-transform: uppercase;
}
.th-react {
    color: #ff4757;
}
.th-vue {
    color: #ffd32a;
}

.rank-table tbody tr {
    transition: background 0.2s ease;
    border-bottom: 1px solid rgba(255, 255, 255, 0.03);
}
.rank-table tbody tr:hover {
    background: rgba(0, 255, 242, 0.04);
}
.rank-table td {
    padding: 9px 12px;
    text-align: center;
    font-size: 14px;
    color: rgba(255, 255, 255, 0.75);
}
.city-name {
    font-weight: 600;
    color: rgba(255, 255, 255, 0.9);
    font-size: 13px;
}

/* ============ 通用 ============ */
.mini-spinner {
    display: inline-block;
    width: 14px;
    height: 14px;
    border: 2px solid rgba(0, 255, 242, 0.15);
    border-top-color: #00fff2;
    border-radius: 50%;
    animation: spin 0.8s linear infinite;
    vertical-align: middle;
}

/* ============ 响应式 ============ */

/* --- 平板竖屏 (≤1024px) --- */
@media (max-width: 1024px) {
    .dashboard {
        padding: 12px;
        gap: 12px;
    }
    .dash-right {
        width: 280px;
    }
    .title-decoration {
        width: 50px;
    }
    .dash-title {
        font-size: 20px;
        letter-spacing: 4px;
    }
}

/* --- 小平板 / 大手机横屏 (≤900px) --- */
@media (max-width: 900px) {
    .dash-body {
        flex-direction: column;
    }
    .dash-right {
        width: 100%;
    }
    .table-panel {
        max-height: 380px;
    }
    .city-cards {
        grid-template-columns: repeat(3, 1fr);
    }
}

/* --- 手机横屏 / 小平板 (≤767px) --- */
@media (max-width: 767px) {
    .dashboard {
        padding: 10px;
        gap: 10px;
    }

    /* 头部重排 */
    .dash-header {
        flex-wrap: wrap;
        gap: 10px;
        padding: 12px 14px;
        min-height: auto;
    }
    .header-center {
        order: -1;
        width: 100%;
        margin-bottom: 2px;
    }
    .header-left {
        order: 0;
    }
    .header-right {
        order: 1;
        margin-left: auto;
    }
    .dash-title {
        font-size: 18px;
        letter-spacing: 3px;
    }
    .dash-subtitle {
        font-size: 10px;
    }
    .title-decoration {
        width: 40px;
    }
    .title-wrapper {
        gap: 12px;
    }
    .neon-btn {
        padding: 7px 16px;
        font-size: 12px;
    }

    /* 图表面板 - 去掉flex:1，用固定高度避免底部空白 */
    .chart-panel {
        flex: none;
        height: 320px;
        min-height: auto;
    }
    .panel-header {
        flex-direction: column;
        align-items: flex-start;
        gap: 10px;
        padding: 12px 14px;
    }
    .city-tabs {
        width: 100%;
        overflow-x: auto;
        -webkit-overflow-scrolling: touch;
        scrollbar-width: none;
        gap: 6px;
        padding-bottom: 2px;
    }
    .city-tabs::-webkit-scrollbar {
        display: none;
    }
    .city-tabs button {
        padding: 6px 14px;
        font-size: 12px;
        white-space: nowrap;
        flex-shrink: 0;
        min-height: 32px;
    }

    /* 数据卡片 - 2列，日期卡片占满一行 */
    .city-cards {
        grid-template-columns: repeat(2, 1fr);
        gap: 10px;
    }
    .city-cards .info-card {
        grid-column: 1 / -1;
        flex-direction: row;
        align-items: center;
        gap: 16px;
        padding: 10px 16px;
    }
    .info-card .card-divider {
        display: none;
    }
    .info-card .card-hint {
        margin-bottom: 0;
    }
    .info-card .card-row {
        margin: 0;
    }
    .stat-card {
        padding: 12px 12px;
    }
    .card-label {
        font-size: 13px;
    }
    .card-value {
        font-size: 20px;
    }
    .card-value.dim {
        font-size: 14px;
    }
    .framework-tag {
        font-size: 9px;
        padding: 2px 6px;
    }
    .card-yesterday {
        font-size: 10px;
    }

    /* 右侧表格 - 不隐藏，改为可滚动面板 */
    .dash-right {
        width: 100%;
    }
    .table-panel {
        max-height: 320px;
    }
    .rank-table th {
        padding: 8px 8px;
        font-size: 11px;
    }
    .rank-table td {
        padding: 8px 8px;
        font-size: 13px;
    }
    .city-name {
        font-size: 12px;
    }

    .lang-label {
        display: none;
    }
}

/* --- 小手机 (≤480px) --- */
@media (max-width: 480px) {
    .dashboard {
        padding: 8px;
        gap: 8px;
        /* 安全区域适配（刘海屏/底部横条） */
        padding-top: calc(8px + env(safe-area-inset-top));
        padding-bottom: calc(8px + env(safe-area-inset-bottom));
        padding-left: calc(8px + env(safe-area-inset-left));
        padding-right: calc(8px + env(safe-area-inset-right));
    }

    .dash-header {
        padding: 10px 12px;
        gap: 8px;
        border-radius: 10px;
    }
    .dash-title {
        font-size: 16px;
        letter-spacing: 2px;
    }
    .title-decoration {
        width: 24px;
    }
    .title-decoration::after {
        width: 4px;
        height: 4px;
    }
    .title-wrapper {
        gap: 8px;
    }
    .dash-subtitle {
        font-size: 9px;
        letter-spacing: 1px;
    }
    .github-badges {
        display: none;
    }
    .neon-btn {
        padding: 6px 12px;
        font-size: 11px;
        border-radius: 5px;
    }
    .feedback-link a {
        font-size: 11px;
        padding: 4px 10px;
    }

    /* 图表 */
    .chart-panel {
        flex: none;
        height: 280px;
        min-height: auto;
        border-radius: 10px;
    }
    .chart-container {
        padding: 4px;
    }
    .panel-header {
        padding: 10px 12px;
        gap: 8px;
    }
    .panel-title {
        font-size: 13px;
        gap: 6px;
    }
    .dot {
        width: 6px;
        height: 6px;
    }

    /* 卡片 - 仍然2列但更紧凑 */
    .city-cards {
        grid-template-columns: repeat(2, 1fr);
        gap: 8px;
    }
    .stat-card {
        padding: 10px 10px;
        border-radius: 10px;
    }
    .card-corner {
        width: 8px;
        height: 8px;
    }
    .card-label {
        font-size: 12px;
        letter-spacing: 1px;
    }
    .card-hint {
        font-size: 9px;
    }
    .card-divider {
        margin: 6px 0;
    }
    .card-row {
        margin: 3px 0;
    }
    .card-value {
        font-size: 18px;
    }
    .card-value.dim {
        font-size: 13px;
    }
    .framework-tag {
        font-size: 8px;
        padding: 1px 5px;
    }
    .card-yesterday {
        font-size: 9px;
    }

    /* 表格面板 */
    .table-panel {
        max-height: 280px;
        border-radius: 10px;
    }
    .rank-table th {
        padding: 7px 6px;
        font-size: 10px;
        letter-spacing: 0;
    }
    .rank-table td {
        padding: 7px 6px;
        font-size: 12px;
    }
    .city-name {
        font-size: 11px;
    }

    /* 玻璃卡片在小屏上减轻模糊 */
    .glass-card {
        backdrop-filter: blur(8px);
        -webkit-backdrop-filter: blur(8px);
        border-radius: 10px;
    }
}

/* --- 超小屏 (≤360px) --- */
@media (max-width: 360px) {
    .dash-title {
        font-size: 14px;
        letter-spacing: 1px;
    }
    .title-decoration {
        display: none;
    }
    .city-cards {
        grid-template-columns: 1fr;
    }
    .stat-card {
        padding: 10px 14px;
    }
    .card-value {
        font-size: 20px;
    }
    .chart-panel {
        height: 240px;
        min-height: auto;
    }
    .neon-btn {
        padding: 6px 10px;
        font-size: 10px;
    }
}

/* --- 横屏手机优化 --- */
@media (max-height: 500px) and (orientation: landscape) {
    .dashboard {
        padding: 6px 12px;
        gap: 8px;
    }
    .dash-header {
        min-height: auto;
        padding: 8px 14px;
    }
    .dash-title {
        font-size: 16px;
    }
    .chart-panel {
        flex: none;
        height: 220px;
        min-height: auto;
    }
    .city-cards {
        grid-template-columns: repeat(auto-fit, minmax(130px, 1fr));
        gap: 8px;
    }
    .stat-card {
        padding: 8px 10px;
    }
    .card-value {
        font-size: 16px;
    }
    .card-label {
        font-size: 11px;
    }
    .table-panel {
        max-height: 260px;
    }
}

/* --- 触摸设备优化 --- */
@media (hover: none) and (pointer: coarse) {
    .city-tabs button {
        min-height: 36px;
        min-width: 44px;
    }
    .neon-btn {
        min-height: 36px;
    }
    .feedback-link a {
        min-height: 32px;
        display: inline-flex;
        align-items: center;
    }
    /* 触摸设备不需要 hover 发光效果，减少性能消耗 */
    .glass-card:hover {
        border-color: rgba(0, 255, 242, 0.12);
        box-shadow:
            0 0 20px rgba(0, 0, 0, 0.3),
            inset 0 1px 0 rgba(255, 255, 255, 0.05);
    }
    .rank-table tbody tr:hover {
        background: transparent;
    }
    /* 增大表格行高方便触摸 */
    .rank-table td {
        padding: 10px 8px;
    }
}
</style>
