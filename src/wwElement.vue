<template>
  <div class="es" ref="rootRef">
    <!-- ═══ TOAST NOTIFICATIONS ═══ -->
    <div class="es__toasts">
      <div v-for="toast in toasts" :key="toast.id" class="es__toast" :class="'es__toast--' + toast.type">
        <div class="es__toast-header">
          <svg v-if="toast.type === 'success'" width="16" height="16" viewBox="0 0 20 20" fill="none"><path d="M16.707 5.293a1 1 0 010 1.414l-8 8a1 1 0 01-1.414 0l-4-4a1 1 0 011.414-1.414L8 12.586l7.293-7.293a1 1 0 011.414 0z" fill="currentColor"/></svg>
          <svg v-else width="16" height="16" viewBox="0 0 20 20" fill="none"><circle cx="10" cy="10" r="8" stroke="currentColor" stroke-width="1.5" fill="none"/><path d="M10 6v5M10 13.5h.01" stroke="currentColor" stroke-width="1.5" stroke-linecap="round"/></svg>
          <span class="es__toast-title">{{ toast.title }}</span>
          <button class="es__toast-close" @click="removeToast(toast.id)">
            <svg width="14" height="14" viewBox="0 0 20 20" fill="none"><path d="M6 6l8 8M14 6l-8 8" stroke="currentColor" stroke-width="2" stroke-linecap="round"/></svg>
          </button>
        </div>
        <div v-if="toast.message" class="es__toast-body">{{ toast.message }}</div>
      </div>
    </div>

    <!-- ═══ ERROR BANNER ═══ -->
    <div v-if="loadError" class="es__error-banner">
      <svg width="16" height="16" viewBox="0 0 20 20" fill="none"><circle cx="10" cy="10" r="8" stroke="currentColor" stroke-width="1.5" fill="none"/><path d="M10 6v5M10 13.5h.01" stroke="currentColor" stroke-width="1.5" stroke-linecap="round"/></svg>
      <div class="es__error-banner-content">
        <strong>Failed to load data</strong>
        <span>{{ loadError }}</span>
      </div>
      <button class="es__error-banner-action" @click="retryLoad">Retry</button>
      <button class="es__error-banner-dismiss" @click="loadError = null">
        <svg width="14" height="14" viewBox="0 0 20 20" fill="none"><path d="M6 6l8 8M14 6l-8 8" stroke="currentColor" stroke-width="2" stroke-linecap="round"/></svg>
      </button>
    </div>

    <div class="es__scroll-area">
    <div class="es__page-header">
      <div class="es__page-header-left">
        <h1 class="es__page-title">{{ content?.pageTitle || 'Conditional Currency Multipliers' }}</h1>
        <p class="es__page-desc">{{ content?.pageDescription || 'Configure reward multipliers based on tier, products, and time conditions.' }}</p>
      </div>
      <div class="es__page-header-right">
        <div class="es__filter-bar">
          <input class="es__search-input" v-model="searchFactorGroup" placeholder="Search earn factor group..." />
          <input class="es__search-input" v-model="searchCondGroup" placeholder="Search condition group..." />
          <select class="es__filter-select" v-model="filterFactorType">
            <option value="">All</option>
            <option value="rate">Earn rate</option>
            <option value="multiplier">Multiplier</option>
          </select>
          <div class="es__create-dropdown" @mouseenter="createMenuOpen = true" @mouseleave="createMenuOpen = false">
            <button class="es__primary-btn" @click="createMenuOpen = !createMenuOpen">
              Create
              <svg width="12" height="12" viewBox="0 0 20 20" fill="none"><path d="M6 8l4 4 4-4" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/></svg>
            </button>
            <div class="es__create-menu" v-show="createMenuOpen">
              <button class="es__create-menu-item" @click="openCreateFactorGroup(); createMenuOpen = false">
                <svg width="14" height="14" viewBox="0 0 20 20" fill="none"><path d="M10 4v12M4 10h12" stroke="currentColor" stroke-width="2" stroke-linecap="round"/></svg>
                Earn Factor Group
              </button>
              <button class="es__create-menu-item" @click="openCreateConditionGroup(); createMenuOpen = false">
                <svg width="14" height="14" viewBox="0 0 20 20" fill="none"><path d="M10 4v12M4 10h12" stroke="currentColor" stroke-width="2" stroke-linecap="round"/></svg>
                Earn Condition Group
              </button>
            </div>
          </div>
        </div>
      </div>
    </div>
    <div class="es__layout" ref="layoutRef">
      <div class="es__header-row">
        <div class="es__col-label es__col-label--left">Earn factor group</div>
        <div class="es__col-label es__col-label--right">Earn conditions group</div>
      </div>

      <div v-if="loadingFactorGroups && !factorGroups?.length" class="es__loading"><div class="es__spinner"></div></div>

      <template v-else>
        <!-- ═══ LINKED GROUPS ═══ -->
        <div v-for="entry in linkedGroupEntries" :key="entry.group.id" class="es__group-row">
          <div class="es__group-left">
            <div class="es__group-sidebar" :style="sidebarStyle(entry.group.id)">
              <div class="es__sidebar-accent" :style="{ background: getGroupColor(entry.group.id) }"></div>
              <div class="es__sidebar-inner">
                <span class="es__sidebar-name">{{ entry.group.name || 'Untitled Group' }}</span>
                <button class="es__sidebar-action" @click="handleAddFactor(entry.group.id)" title="Add earn factor">
                  <svg width="12" height="12" viewBox="0 0 20 20" fill="none"><path d="M10 4v12M4 10h12" stroke="currentColor" stroke-width="2.5" stroke-linecap="round"/></svg>
                </button>
                <button class="es__sidebar-action" @click="handleEditFactorGroup(entry.group)" title="Edit group">
                  <svg width="12" height="12" viewBox="0 0 20 20" fill="none"><path d="M13.586 3.586a2 2 0 112.828 2.828l-.793.793-2.828-2.828.793-.793zM11.379 5.793L3 14.172V17h2.828l8.38-8.379-2.83-2.828z" fill="currentColor"/></svg>
                </button>
              </div>
            </div>
            <div class="es__group-cards">
              <div v-for="f in entry.factors" :key="f.id" :data-factor-id="f.id" class="es__card es__card--factor">
                <div class="es__card-accent" :style="{ background: getGroupColor(f.earn_factor_group_id) }"></div>
                <div class="es__card-body">
                  <div class="es__card-icon" :class="f.target_currency === 'ticket' ? 'es__card-icon--credit' : 'es__card-icon--points'">
                    <svg v-if="f.earn_factor_type === 'rate'" width="14" height="14" viewBox="0 0 20 20" fill="none"><path d="M3 3h6l8 8-6 6-8-8V3z" stroke="currentColor" stroke-width="1.5" stroke-linejoin="round"/><circle cx="7" cy="7" r="1.5" fill="currentColor"/></svg>
                    <svg v-else width="14" height="14" viewBox="0 0 20 20" fill="none"><path d="M11 2L4 11h5l-1 7 7-9h-5l1-7z" fill="currentColor"/></svg>
                  </div>
                  <div class="es__card-info">
                    <div class="es__card-name">{{ getFactorTitle(f) }}</div>
                    <div class="es__card-sub"><span>{{ getFactorType(f) }}</span></div>
                  </div>
                  <span v-if="f.earn_factor_type === 'multiplier'" class="es__card-badge es__card-badge--mult">{{ f.earn_factor_amount || 0 }}x</span>
                  <span v-else-if="f.earn_factor_amount" class="es__card-badge es__card-badge--rate">฿{{ f.earn_factor_amount }}</span>
                  <button class="es__card-edit" @click="handleEditFactor(f)">
                    <svg width="14" height="14" viewBox="0 0 20 20" fill="none"><path d="M13.586 3.586a2 2 0 112.828 2.828l-.793.793-2.828-2.828.793-.793zM11.379 5.793L3 14.172V17h2.828l8.38-8.379-2.83-2.828z" fill="currentColor"/></svg>
                  </button>
                </div>
                <button class="es__card-connect" @click="handleConnectFactor(f, $event)">
                  <svg width="12" height="12" viewBox="0 0 20 20" fill="none"><path d="M10 4v12M4 10h12" stroke="currentColor" stroke-width="2.5" stroke-linecap="round"/></svg>
                </button>
              </div>
            </div>
          </div>
          <div class="es__group-right">
            <div class="es__cond-slots">
              <div v-for="f in entry.factors" :key="f.id" class="es__cond-slot">
                <div v-if="f._condGroupInfo" :data-cg-key="f._dk"
                  class="es__card es__card--condition" :class="{ 'es__card--expanded': expandedConds[f._dk] }">
                  <div class="es__card-dot-indicator"></div>
                  <div class="es__cond-wrap">
                    <div class="es__cond-header" @click="handleEditConditionGroup(f._condGroupInfo.group)">
                      <div class="es__cg-icon" :style="cgIconStyle(f._condGroupInfo.group?.id)">
                        <svg width="14" height="14" viewBox="0 0 20 20" fill="none"><path d="M3 5h14M6 10h8M8 15h4" stroke="currentColor" stroke-width="1.8" stroke-linecap="round"/></svg>
                      </div>
                      <div class="es__card-info">
                        <div class="es__card-name">{{ f._condGroupInfo.group?.name || 'Untitled' }}</div>
                        <div class="es__card-sub">
                          <span>{{ f._condGroupInfo.conditions?.length || 0 }} condition{{ (f._condGroupInfo.conditions?.length || 0) !== 1 ? 's' : '' }}</span>
                          <template v-if="f._condGroupInfo.linkedCount > 0">
                            <span class="es__linked-badge">
                              <svg width="14" height="14" viewBox="0 0 14 14" fill="none"><path d="M9.5 4.5V3.5a2 2 0 00-2-2h-1a2 2 0 00-2 2v1M4.5 9.5v1a2 2 0 002 2h1a2 2 0 002-2v-1M7 4.5v5" stroke="currentColor" stroke-width="1.3" stroke-linecap="round" stroke-linejoin="round"/></svg>
                              {{ f._condGroupInfo.linkedCount }}
                            </span>
                          </template>
                        </div>
                      </div>
                      <button class="es__cond-edit" @click.stop="handleEditConditionGroup(f._condGroupInfo.group)" title="Edit condition group">
                        <svg width="14" height="14" viewBox="0 0 20 20" fill="none"><path d="M13.586 3.586a2 2 0 112.828 2.828l-.793.793-2.828-2.828.793-.793zM11.379 5.793L3 14.172V17h2.828l8.38-8.379-2.83-2.828z" fill="currentColor"/></svg>
                      </button>
                      <button class="es__expand-btn" @click.stop="toggleCondExpand(f._dk)"
                        :class="{ 'es__expand-btn--open': expandedConds[f._dk] }">
                        <svg width="12" height="12" viewBox="0 0 20 20" fill="none"><path d="M6 12.5l4-5 4 5" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/></svg>
                      </button>
                    </div>
                    <div v-if="expandedConds[f._dk] && f._condGroupInfo.conditions?.length" class="es__cond-detail">
                      <div class="es__cond-table-wrap">
                        <table class="es__cond-table">
                          <colgroup>
                            <col class="es__col-type" />
                            <col class="es__col-items" />
                            <col class="es__col-logic" />
                            <col class="es__col-threshold" />
                            <col class="es__col-excess" />
                          </colgroup>
                          <thead><tr><th>Type</th><th>Items</th><th>Logic</th><th>Threshold type</th><th>Excess</th></tr></thead>
                          <tbody>
                            <tr v-for="(cond, ci) in f._condGroupInfo.conditions" :key="ci">
                              <td>{{ formatEntity(cond?.entity) }}</td>
                              <td>
                                <span class="es__items-badge">
                                  {{ cond?.entity_ids?.length || 0 }}
                                  <svg width="12" height="12" viewBox="0 0 16 16" fill="none"><ellipse cx="8" cy="8" rx="4.5" ry="3" stroke="currentColor" stroke-width="1"/><circle cx="8" cy="8" r="1.8" fill="currentColor"/><path d="M1.5 8c1.5-3.5 5-5 6.5-5s5 1.5 6.5 5c-1.5 3.5-5 5-6.5 5s-5-1.5-6.5-5z" stroke="currentColor" stroke-width="1" fill="none"/></svg>
                                </span>
                              </td>
                              <td>{{ cond?.operator || '-' }}</td>
                              <td>
                                <span class="es__threshold-cell">
                                  <span>{{ isProductEntityType(cond?.entity) ? formatThreshold(cond?.threshold_unit) : '—' }}</span>
                                  <svg v-if="isProductEntityType(cond?.entity) && cond?.threshold_unit" width="14" height="14" viewBox="0 0 16 16" fill="none"><rect x="3" y="1.5" width="10" height="13" rx="1.5" stroke="currentColor" stroke-width="1" fill="none"/><path d="M6 5.5h4M6 8h3" stroke="currentColor" stroke-width="0.9" stroke-linecap="round"/></svg>
                                </span>
                              </td>
                              <td>{{ isProductEntityType(cond?.entity) && cond?.threshold_unit ? (cond?.apply_to_excess_only ? 'Yes' : 'No') : '—' }}</td>
                            </tr>
                          </tbody>
                        </table>
                      </div>
                    </div>
                  </div>
                </div>
              </div>
            </div>
          </div>
        </div>

        <!-- ═══ UNLINKED SECTION ═══ -->
        <template v-if="hasUnlinkedSection">
          <div class="es__divider"><span>Unlinked</span></div>
          <div class="es__unlinked-row">
            <div class="es__unlinked-left">
              <div v-for="entry in unlinkedGroupsWithFactors" :key="entry.group.id" class="es__unlinked-group">
                <div class="es__group-sidebar" :style="sidebarStyle(entry.group.id)">
                  <div class="es__sidebar-accent" :style="{ background: getGroupColor(entry.group.id) }"></div>
                  <div class="es__sidebar-inner">
                    <span class="es__sidebar-name">{{ entry.group.name || 'Untitled Group' }}</span>
                    <button class="es__sidebar-action" @click="handleAddFactor(entry.group.id)" title="Add earn factor">
                      <svg width="12" height="12" viewBox="0 0 20 20" fill="none"><path d="M10 4v12M4 10h12" stroke="currentColor" stroke-width="2.5" stroke-linecap="round"/></svg>
                    </button>
                    <button class="es__sidebar-action" @click="handleEditFactorGroup(entry.group)" title="Edit group">
                      <svg width="12" height="12" viewBox="0 0 20 20" fill="none"><path d="M13.586 3.586a2 2 0 112.828 2.828l-.793.793-2.828-2.828.793-.793zM11.379 5.793L3 14.172V17h2.828l8.38-8.379-2.83-2.828z" fill="currentColor"/></svg>
                    </button>
                  </div>
                </div>
                <div class="es__group-cards">
                  <div v-for="f in entry.factors" :key="f.id" :data-factor-id="f.id" class="es__card es__card--factor es__card--dim">
                    <div class="es__card-accent" :style="{ background: getGroupColor(f.earn_factor_group_id) }"></div>
                    <div class="es__card-body">
                      <div class="es__card-icon" :class="f.target_currency === 'ticket' ? 'es__card-icon--credit' : 'es__card-icon--points'">
                        <svg v-if="f.earn_factor_type === 'rate'" width="14" height="14" viewBox="0 0 20 20" fill="none"><path d="M3 3h6l8 8-6 6-8-8V3z" stroke="currentColor" stroke-width="1.5" stroke-linejoin="round"/><circle cx="7" cy="7" r="1.5" fill="currentColor"/></svg>
                        <svg v-else width="14" height="14" viewBox="0 0 20 20" fill="none"><path d="M11 2L4 11h5l-1 7 7-9h-5l1-7z" fill="currentColor"/></svg>
                      </div>
                      <div class="es__card-info">
                        <div class="es__card-name">{{ getFactorTitle(f) }}</div>
                        <div class="es__card-sub"><span>{{ getFactorType(f) }}</span></div>
                      </div>
                      <span v-if="f.earn_factor_type === 'multiplier'" class="es__card-badge es__card-badge--mult">{{ f.earn_factor_amount || 0 }}x</span>
                      <span v-else-if="f.earn_factor_amount" class="es__card-badge es__card-badge--rate">฿{{ f.earn_factor_amount }}</span>
                      <button class="es__card-edit" @click="handleEditFactor(f)">
                        <svg width="14" height="14" viewBox="0 0 20 20" fill="none"><path d="M13.586 3.586a2 2 0 112.828 2.828l-.793.793-2.828-2.828.793-.793zM11.379 5.793L3 14.172V17h2.828l8.38-8.379-2.83-2.828z" fill="currentColor"/></svg>
                      </button>
                    </div>
                    <button class="es__card-connect" @click="handleConnectFactor(f, $event)">
                      <svg width="12" height="12" viewBox="0 0 20 20" fill="none"><path d="M10 4v12M4 10h12" stroke="currentColor" stroke-width="2.5" stroke-linecap="round"/></svg>
                    </button>
                  </div>
                </div>
              </div>

              <div v-for="entry in emptyGroups" :key="entry.group.id" class="es__unlinked-group es__unlinked-group--empty">
                <div class="es__group-sidebar" :style="sidebarStyle(entry.group.id)">
                  <div class="es__sidebar-accent" :style="{ background: getGroupColor(entry.group.id) }"></div>
                  <div class="es__sidebar-inner">
                    <span class="es__sidebar-name">{{ entry.group.name || 'Untitled Group' }}</span>
                    <button class="es__sidebar-action" @click="handleAddFactor(entry.group.id)" title="Add earn factor">
                      <svg width="12" height="12" viewBox="0 0 20 20" fill="none"><path d="M10 4v12M4 10h12" stroke="currentColor" stroke-width="2.5" stroke-linecap="round"/></svg>
                    </button>
                    <button class="es__sidebar-action" @click="handleEditFactorGroup(entry.group)" title="Edit group">
                      <svg width="12" height="12" viewBox="0 0 20 20" fill="none"><path d="M13.586 3.586a2 2 0 112.828 2.828l-.793.793-2.828-2.828.793-.793zM11.379 5.793L3 14.172V17h2.828l8.38-8.379-2.83-2.828z" fill="currentColor"/></svg>
                    </button>
                  </div>
                </div>
              </div>
            </div>

            <div class="es__unlinked-right" v-if="unlinkedCondGroups.length">
              <div v-for="cg in unlinkedCondGroups" :key="cg.id" class="es__cond-slot">
                <div class="es__card es__card--condition es__card--dim"
                  :class="{ 'es__card--expanded': expandedConds[cg.id] }" :data-cg-key="cg.id">
                  <div class="es__card-dot-indicator"></div>
                  <div class="es__cond-wrap">
                    <div class="es__cond-header" @click="handleEditConditionGroup(cg)">
                      <div class="es__cg-icon" :style="cgIconStyle(cg?.id)">
                        <svg width="14" height="14" viewBox="0 0 20 20" fill="none"><path d="M3 5h14M6 10h8M8 15h4" stroke="currentColor" stroke-width="1.8" stroke-linecap="round"/></svg>
                      </div>
                      <div class="es__card-info">
                        <div class="es__card-name">{{ cg?.name || 'Untitled' }}</div>
                        <div class="es__card-sub">
                          <span>{{ cg._conditions?.length || 0 }} condition{{ (cg._conditions?.length || 0) !== 1 ? 's' : '' }}</span>
                        </div>
                      </div>
                      <button class="es__cond-edit" @click.stop="handleEditConditionGroup(cg)" title="Edit condition group">
                        <svg width="14" height="14" viewBox="0 0 20 20" fill="none"><path d="M13.586 3.586a2 2 0 112.828 2.828l-.793.793-2.828-2.828.793-.793zM11.379 5.793L3 14.172V17h2.828l8.38-8.379-2.83-2.828z" fill="currentColor"/></svg>
                      </button>
                      <button class="es__expand-btn" @click.stop="toggleCondExpand(cg.id)"
                        :class="{ 'es__expand-btn--open': expandedConds[cg.id] }">
                        <svg width="12" height="12" viewBox="0 0 20 20" fill="none"><path d="M6 12.5l4-5 4 5" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/></svg>
                      </button>
                    </div>
                    <div v-if="expandedConds[cg.id] && cg._conditions?.length" class="es__cond-detail">
                      <div class="es__cond-table-wrap">
                        <table class="es__cond-table">
                          <colgroup>
                            <col class="es__col-type" />
                            <col class="es__col-items" />
                            <col class="es__col-logic" />
                            <col class="es__col-threshold" />
                            <col class="es__col-excess" />
                          </colgroup>
                          <thead><tr><th>Type</th><th>Items</th><th>Logic</th><th>Threshold type</th><th>Excess</th></tr></thead>
                          <tbody>
                            <tr v-for="(cond, ci) in cg._conditions" :key="ci">
                              <td>{{ formatEntity(cond?.entity) }}</td>
                              <td>
                                <span class="es__items-badge">
                                  {{ cond?.entity_ids?.length || 0 }}
                                  <svg width="12" height="12" viewBox="0 0 16 16" fill="none"><ellipse cx="8" cy="8" rx="4.5" ry="3" stroke="currentColor" stroke-width="1"/><circle cx="8" cy="8" r="1.8" fill="currentColor"/><path d="M1.5 8c1.5-3.5 5-5 6.5-5s5 1.5 6.5 5c-1.5 3.5-5 5-6.5 5s-5-1.5-6.5-5z" stroke="currentColor" stroke-width="1" fill="none"/></svg>
                                </span>
                              </td>
                              <td>{{ cond?.operator || '-' }}</td>
                              <td>
                                <span class="es__threshold-cell">
                                  <span>{{ isProductEntityType(cond?.entity) ? formatThreshold(cond?.threshold_unit) : '—' }}</span>
                                  <svg v-if="isProductEntityType(cond?.entity) && cond?.threshold_unit" width="14" height="14" viewBox="0 0 16 16" fill="none"><rect x="3" y="1.5" width="10" height="13" rx="1.5" stroke="currentColor" stroke-width="1" fill="none"/><path d="M6 5.5h4M6 8h3" stroke="currentColor" stroke-width="0.9" stroke-linecap="round"/></svg>
                                </span>
                              </td>
                              <td>{{ isProductEntityType(cond?.entity) && cond?.threshold_unit ? (cond?.apply_to_excess_only ? 'Yes' : 'No') : '—' }}</td>
                            </tr>
                          </tbody>
                        </table>
                      </div>
                    </div>
                  </div>
                </div>
              </div>
            </div>
          </div>
        </template>

        <div v-if="!factorGroups?.length" class="es__empty">
          <div class="es__empty-icon">🪙</div>
          <h3 class="es__empty-heading">No earn factors yet</h3>
          <p class="es__empty-desc">Create your first earn factor to start rewarding customers.</p>
          <button class="es__empty-action" @click="openCreateFactorGroup()">
            <svg width="14" height="14" viewBox="0 0 20 20" fill="none"><path d="M10 4v12M4 10h12" stroke="currentColor" stroke-width="2" stroke-linecap="round"/></svg>
            Create Earn Factor Group
          </button>
        </div>
      </template>

      <svg class="es__svg" ref="svgRef">
        <path v-for="ln in lines" :key="ln.key" :d="ln.d" fill="none"
          :stroke="hoveredLine === ln.key ? (content?.connectionLineActiveColor || '#005BD3') : (content?.connectionLineColor || '#C9CCCF')"
          :stroke-width="hoveredLine === ln.key ? 2.5 : 1.5"
          @mouseenter="hoveredLine = ln.key" @mouseleave="hoveredLine = null" style="cursor:pointer" />
      </svg>
    </div>
    <CreateGroupModal v-if="showModal" :type="modalType" :group="editingFactorGroupData"
      @close="showModal = false" @save="handleModalSave" @delete="handleDeleteFactorGroup" />
    <ConnectPopup v-if="connectPopup.open" :condition-groups="allCondGroups" :position="connectPopup.pos"
      @close="connectPopup.open = false" @select="handleConnectSelect" />
    </div>

    <div v-if="panel" class="es__panel-backdrop" @click="panel = null"></div>
    <EarnFactorConfig v-if="panel === 'factor'" class="es__config-panel" :factor="editingFactor" :group-id="editingGroupId"
      :condition-groups="allCondGroups" :ticket-types="ticketTypes" :panel-width="content?.configPanelWidth || '400px'"
      @close="panel = null" @save="saveFactorConfig" @delete="handleDeleteFactor" />
    <EarnConditionGroupConfig v-if="panel === 'condition'" class="es__config-panel" :group="editingCondGroup"
      :all-entity-options="entityOptions" :panel-width="content?.configPanelWidth || '400px'"
      @close="panel = null" @save="saveCondGroupConfig" @delete="handleDeleteCondGroup" />
  </div>
</template>

<script>
import { ref, computed, onMounted, watch, nextTick, onBeforeUnmount } from 'vue';
import { useApi } from './useApi.js';
import EarnFactorConfig from './components/EarnFactorConfig.vue';
import EarnConditionGroupConfig from './components/EarnConditionGroupConfig.vue';
import CreateGroupModal from './components/CreateGroupModal.vue';
import ConnectPopup from './components/ConnectPopup.vue';

// Polaris semantic palette: info, critical, caution, success, decorative-purple, decorative-teal, warning, decorative-indigo
const GROUP_COLORS = ['#2C6ECB', '#D82C0D', '#8A6116', '#29845A', '#6D28D9', '#0D9488', '#C05717', '#4F46E5'];
const ENTITY_LABELS = { product_product: 'Product', product_sku: 'SKU', product_brand: 'Brand', product_category: 'Category', store: 'Store', store_attribute_set: 'Store Attr', tier: 'Tier', persona: 'Persona' };
const THRESHOLD_LABELS = { purchase_amount: 'Purchase amount', purchase_quantity: 'Purchase qty', purchase_count: 'Purchase count', amount: 'Amount', quantity_primary: 'Qty (primary)', quantity_secondary: 'Qty (secondary)' };
const PRODUCT_ENTITIES = new Set(['product_product', 'product_sku', 'product_brand', 'product_category']);

function hashStr(s) { let h = 0; for (let i = 0; i < (s||'').length; i++) { h = ((h<<5)-h)+s.charCodeAt(i); h |= 0; } return Math.abs(h); }

let toastId = 0;

export default {
  components: { EarnFactorConfig, EarnConditionGroupConfig, CreateGroupModal, ConnectPopup },
  props: {
    uid: { type: String, required: true },
    content: { type: Object, required: true },
    /* wwEditor:start */
    wwEditorState: { type: Object, required: true },
    /* wwEditor:end */
  },
  emits: ['trigger-event'],
  setup(props, { emit }) {
    const api = useApi(props);
    const rootRef = ref(null), layoutRef = ref(null), svgRef = ref(null);

    const factorGroups = ref([]);
    const factorsByGroup = ref({});
    const allCondGroups = ref([]);
    const condCache = ref({});
    const entityOptions = ref([]);
    const ticketTypes = ref([]);
    const loadingFactorGroups = ref(false);
    const loadingConditionGroups = ref(false);
    const panel = ref(null);
    const editingFactor = ref(null);
    const editingGroupId = ref(null);
    const editingCondGroup = ref(null);
    const editingFactorGroupData = ref(null);
    const showModal = ref(false);
    const modalType = ref('factor');
    const hoveredLine = ref(null);
    const connectPopup = ref({ open: false, pos: null, factorId: null, groupId: null });
    const lines = ref([]);
    const expandedConds = ref({});
    const searchFactorGroup = ref('');
    const searchCondGroup = ref('');
    const filterFactorType = ref('');
    const createMenuOpen = ref(false);
    const toasts = ref([]);
    const loadError = ref(null);
    let lineTimer = null, ro = null;

    function addToast(type, title, message) {
      const id = ++toastId;
      toasts.value.push({ id, type, title, message });
      setTimeout(() => removeToast(id), 5000);
    }
    function removeToast(id) {
      toasts.value = toasts.value.filter(t => t.id !== id);
    }

    const groupNameMap = computed(() => {
      const m = {};
      for (const g of factorGroups.value || []) { if (g?.id) m[g.id] = g.name || 'Untitled Group'; }
      return m;
    });

    const groupColorMap = computed(() => {
      const map = {};
      let idx = 0;
      for (const g of factorGroups.value || []) {
        if (g?.id && !map[g.id]) { map[g.id] = GROUP_COLORS[idx % GROUP_COLORS.length]; idx++; }
      }
      for (const g of allCondGroups.value || []) {
        if (g?.id && !map[g.id]) { map[g.id] = GROUP_COLORS[idx % GROUP_COLORS.length]; idx++; }
      }
      return map;
    });
    function getGroupColor(gid) { return groupColorMap.value[gid] || GROUP_COLORS[hashStr(gid) % GROUP_COLORS.length]; }
    function getGroupName(gid) { return groupNameMap.value[gid] || 'Unknown Group'; }
    function cgIconStyle(gid) { const c = getGroupColor(gid); return { background: `${c}12`, color: c }; }
    function sidebarStyle(gid) { const c = getGroupColor(gid); return { background: `${c}06` }; }
    function formatEntity(e) { return ENTITY_LABELS[e] || e?.replace(/_/g, ' ')?.replace(/\b\w/g, c => c.toUpperCase()) || '-'; }
    function formatThreshold(u) { return THRESHOLD_LABELS[u] || u?.replace(/_/g, ' ')?.replace(/\b\w/g, c => c.toUpperCase()) || '-'; }
    function isProductEntityType(entity) { return PRODUCT_ENTITIES.has(entity); }

    function toggleCondExpand(dk) {
      expandedConds.value = { ...expandedConds.value, [dk]: !expandedConds.value[dk] };
      scheduleLineUpdate();
      setTimeout(scheduleLineUpdate, 300);
    }

    const groupedEntries = computed(() => {
      const condGroups = allCondGroups.value || [];
      const allF = Object.values(factorsByGroup.value || {}).flat();
      const fgq = searchFactorGroup.value?.toLowerCase()?.trim() || '';
      const ftf = filterFactorType.value || '';
      return (factorGroups.value || []).filter(g => g?.id).filter(g => {
        if (fgq && !(g.name || '').toLowerCase().includes(fgq)) return false;
        return true;
      }).map(g => ({
        group: g,
        factors: (factorsByGroup.value[g.id] || []).filter(f => {
          if (ftf && f?.earn_factor_type !== ftf) return false;
          return true;
        }).map(f => {
          const cid = f?.earn_conditions_group_id;
          let _condGroupInfo = null;
          if (cid) {
            const cg = condGroups.find(x => x?.id === cid);
            if (cg) {
              _condGroupInfo = {
                group: cg,
                conditions: condCache.value[cid]?.conditions || cg?.conditions || [],
                linkedCount: allF.filter(x => x?.earn_conditions_group_id === cid).length,
              };
            }
          }
          return { ...f, _condGroupInfo, _dk: cid ? `${cid}__${f.id}` : null };
        }),
      }));
    });

    const linkedGroupEntries = computed(() => groupedEntries.value.filter(e => e.factors.some(f => f._condGroupInfo)));
    const unlinkedGroupsWithFactors = computed(() => groupedEntries.value.filter(e => e.factors.length > 0 && !e.factors.some(f => f._condGroupInfo)));
    const emptyGroups = computed(() => groupedEntries.value.filter(e => e.factors.length === 0));
    const hasUnlinkedSection = computed(() => unlinkedGroupsWithFactors.value.length > 0 || emptyGroups.value.length > 0 || unlinkedCondGroups.value.length > 0);
    const unlinkedCondGroups = computed(() => {
      const allF = Object.values(factorsByGroup.value || {}).flat();
      const linked = new Set(allF.map(f => f?.earn_conditions_group_id).filter(Boolean));
      const cgq = searchCondGroup.value?.toLowerCase()?.trim() || '';
      return (allCondGroups.value || []).filter(g => {
        if (!g?.id || linked.has(g.id)) return false;
        if (cgq && !(g.name || '').toLowerCase().includes(cgq)) return false;
        return true;
      }).map(g => ({
        ...g, _conditions: condCache.value[g.id]?.conditions || g?.conditions || [],
      }));
    });

    async function loadAll() {
      loadError.value = null;
      await Promise.all([loadFactorGroups(), loadCondGroups(), loadEntities(), loadTicketTypes()]);
      emit('trigger-event', { name: 'data-loaded', event: { factorGroupCount: factorGroups.value?.length, conditionGroupCount: allCondGroups.value?.length } });
      scheduleLineUpdate(); setTimeout(scheduleLineUpdate, 300); setTimeout(scheduleLineUpdate, 800);
    }
    async function loadFactorGroups() {
      loadingFactorGroups.value = true;
      try {
        factorGroups.value = await api.fetchEarnFactorGroups() || [];
        const m = {};
        for (const g of factorGroups.value) { if (!g?.id) continue; const factors = await api.fetchFactorsByGroup(g.id) || []; m[g.id] = factors.map(f => ({ ...f, earn_factor_group_id: f.earn_factor_group_id || g.id })); }
        factorsByGroup.value = m;
      } catch (e) { loadError.value = e?.message || 'Failed to load earn factor groups'; errMsg('Load failed', e); } finally { loadingFactorGroups.value = false; }
    }
    async function loadCondGroups() {
      loadingConditionGroups.value = true;
      try { allCondGroups.value = await api.fetchAllConditionGroups() || []; const c = {}; for (const g of allCondGroups.value) { if (g?.id) try { c[g.id] = await api.fetchConditionGroupDetails(g.id); } catch {} } condCache.value = c; }
      catch (e) { loadError.value = e?.message || 'Failed to load condition groups'; errMsg('Load failed', e); } finally { loadingConditionGroups.value = false; }
    }
    async function loadEntities() { try { entityOptions.value = await api.fetchEntityOptions() || []; } catch (e) { errMsg('Load failed', e); } }
    async function loadTicketTypes() { try { ticketTypes.value = await api.fetchTicketTypes() || []; } catch (e) { console.warn('Ticket types load failed', e); } }
    function errMsg(m, e) { console.error(m, e); emit('trigger-event', { name: 'error', event: { message: m, code: 'ERR' } }); }
    function retryLoad() { loadError.value = null; loadAll(); }

    function scheduleLineUpdate() { clearTimeout(lineTimer); lineTimer = setTimeout(() => nextTick(rebuildLines), 150); }

    function rebuildLines() {
      const root = rootRef.value; if (!root) return;
      const layout = layoutRef.value; if (!layout) return;
      const lr = layout.getBoundingClientRect();
      const newLines = [];
      for (const entry of linkedGroupEntries.value) {
        for (const f of entry.factors) {
          if (!f._dk) continue;
          const doc = typeof wwLib !== 'undefined' ? wwLib.getFrontDocument() : document;
          const fEl = root.querySelector(`[data-factor-id="${f.id}"]`) || doc.querySelector(`[data-factor-id="${f.id}"]`);
          const cEl = root.querySelector(`[data-cg-key="${f._dk}"]`) || doc.querySelector(`[data-cg-key="${f._dk}"]`);
          if (!fEl || !cEl) continue;
          const fR = fEl.getBoundingClientRect(); const cR = cEl.getBoundingClientRect();
          const x1 = fR.right - lr.left; const y1 = fR.top + fR.height / 2 - lr.top;
          const x2 = cR.left - lr.left; const y2 = cR.top + 30 - lr.top;
          if (x2 <= x1) continue;
          const cp = Math.max((x2 - x1) * 0.4, 40);
          newLines.push({ key: `${f.id}__${f.earn_conditions_group_id}`, d: `M ${x1} ${y1} C ${x1 + cp} ${y1}, ${x2 - cp} ${y2}, ${x2} ${y2}` });
        }
      }
      lines.value = newLines;
    }

    function getFactorTitle(f) { if (!f) return 'Unnamed'; if (f.name || f.target_entity_name) return f.name || f.target_entity_name; const c = f.target_currency === 'ticket' ? 'Credit' : 'Points'; return f.earn_factor_type === 'rate' ? `${c} Starter Rules` : `${c} Power Boost`; }
    function getFactorType(f) { if (!f) return ''; const c = f.target_currency === 'ticket' ? 'Store credit' : 'Points'; return `${c} (${f.earn_factor_type === 'rate' ? 'Base rate' : 'Multiplier'})`; }

    function handleAddFactor(groupId) { editingFactor.value = { earn_factor_type: 'rate', public: true, target_currency: 'points', active_status: true }; editingGroupId.value = groupId; panel.value = 'factor'; }
    function handleEditFactor(f) { editingFactor.value = { ...f }; editingGroupId.value = f?.earn_factor_group_id; panel.value = 'factor'; }
    function handleEditFactorGroup(g) { editingFactorGroupData.value = g || null; modalType.value = 'factor'; showModal.value = true; }
    async function handleEditConditionGroup(g) { try { editingCondGroup.value = g?.id ? await api.fetchConditionGroupDetails(g.id) : { id: null, name: '', conditions: [] }; } catch { editingCondGroup.value = { ...g, conditions: g?.conditions || [] }; } panel.value = 'condition'; }
    function openCreateFactorGroup() { editingFactorGroupData.value = null; modalType.value = 'factor'; showModal.value = true; }
    function openCreateConditionGroup() { editingCondGroup.value = null; panel.value = 'condition'; }

    async function handleModalSave(p) {
      try {
        if (modalType.value === 'factor') {
          const payload = { name: p.name, stackable: p.stackable, window_start: p.window_start, window_end: p.window_end };
          if (p.id) payload.id = p.id;
          if (!p.id) payload.factors = [];
          await api.upsertEarnFactorGroup(payload);
          await loadFactorGroups();
          const action = p.id ? 'updated' : 'created';
          addToast('success', `Earn Factor Group ${action === 'created' ? 'Created' : 'Updated'}`, `Earn factor group "${p.name}" ${action} successfully`);
          emit('trigger-event', { name: 'earn-factor-group-saved', event: { groupId: p.id, groupName: p.name, action } });
        }
      } catch (e) {
        errMsg('Save failed', e);
        addToast('error', 'Save Failed', e?.message || 'Failed to save earn factor group');
      }
      showModal.value = false;
      scheduleLineUpdate();
    }

    async function saveFactorConfig({ groupId, factor }) {
      try {
        const det = await api.fetchEarnFactorGroupDetails(groupId);
        const ex = det?.factors || [];
        const up = factor.id ? ex.map(f => f.id === factor.id ? factor : f) : [...ex, factor];
        await api.upsertEarnFactorGroup({ id: groupId, factors: up });
        await loadFactorGroups();
        panel.value = null;
        const action = factor.id ? 'updated' : 'created';
        addToast('success', `Earn Factor ${action === 'created' ? 'Created' : 'Updated'}`, `Earn factor "${factor.name || 'Unnamed'}" ${action} successfully`);
        emit('trigger-event', { name: 'earn-factor-saved', event: { factorId: factor.id, factorType: factor.earn_factor_type, groupId } });
      } catch (e) {
        errMsg('Save failed', e);
        addToast('error', 'Save Failed', e?.message || 'Failed to save earn factor');
      }
      scheduleLineUpdate();
    }

    async function saveCondGroupConfig(payload) {
      try {
        await api.upsertConditionGroup(payload);
        await loadCondGroups();
        panel.value = null;
        const action = payload.id ? 'updated' : 'created';
        addToast('success', `Earn Condition Group ${action === 'created' ? 'Created' : 'Updated'}`, `Earn condition group "${payload.name || 'Unnamed'}" ${action} successfully`);
        emit('trigger-event', { name: 'earn-condition-group-saved', event: { groupId: payload.id, groupName: payload.name, action } });
      } catch (e) {
        errMsg('Save failed', e);
        addToast('error', 'Save Failed', e?.message || 'Failed to save earn condition group');
      }
      scheduleLineUpdate();
    }

    async function handleDeleteFactor({ factorId, groupId }) {
      try {
        await api.deleteEarnFactor(factorId);
        await loadFactorGroups();
        panel.value = null;
        addToast('success', 'Earn Factor Deleted', 'Earn factor deleted successfully');
      } catch (e) {
        errMsg('Delete failed', e);
        addToast('error', 'Delete Failed', e?.message || 'Failed to delete earn factor');
      }
      scheduleLineUpdate();
    }

    async function handleDeleteFactorGroup({ groupId }) {
      try {
        const factors = factorsByGroup.value[groupId] || [];
        for (const f of factors) {
          if (f?.id) await api.deleteEarnFactor(f.id);
        }
        await api.deleteEarnFactorGroup(groupId);
        await loadFactorGroups();
        showModal.value = false;
        addToast('success', 'Earn Factor Group Deleted', 'Earn factor group deleted successfully');
      } catch (e) {
        errMsg('Delete failed', e);
        addToast('error', 'Delete Failed', e?.message || 'Failed to delete earn factor group');
      }
      scheduleLineUpdate();
    }

    async function handleDeleteCondGroup({ groupId }) {
      try {
        const allF = Object.values(factorsByGroup.value || {}).flat();
        const linkedFactors = allF.filter(f => f?.earn_conditions_group_id === groupId);
        for (const f of linkedFactors) {
          const gid = f.earn_factor_group_id;
          if (!gid) continue;
          const det = await api.fetchEarnFactorGroupDetails(gid);
          const updatedFactors = (det?.factors || []).map(df =>
            df.id === f.id ? { ...df, earn_conditions_group_id: null } : df
          );
          await api.upsertEarnFactorGroup({ id: gid, factors: updatedFactors });
        }
        await api.deleteConditionGroup(groupId);
        await loadFactorGroups();
        await loadCondGroups();
        panel.value = null;
        addToast('success', 'Condition Group Deleted', 'Earn condition group deleted successfully');
      } catch (e) {
        errMsg('Delete failed', e);
        addToast('error', 'Delete Failed', e?.message || 'Failed to delete earn condition group');
      }
      scheduleLineUpdate();
    }

    function handleConnectFactor(f, ev) { const r = ev?.target?.closest?.('.es__card-connect')?.getBoundingClientRect?.() || ev?.target?.getBoundingClientRect?.(); connectPopup.value = { open: true, pos: r ? { x: r.right + 8, y: r.top } : { x: 400, y: 200 }, factorId: f.id, groupId: f.earn_factor_group_id }; }
    async function handleConnectSelect(cg) {
      const { factorId, groupId } = connectPopup.value;
      connectPopup.value.open = false;
      try {
        const det = await api.fetchEarnFactorGroupDetails(groupId);
        const fs = (det?.factors || []).map(f => f.id === factorId ? { ...f, earn_conditions_group_id: cg.id } : f);
        await api.upsertEarnFactorGroup({ id: groupId, factors: fs });
        await loadFactorGroups(); await loadCondGroups();
        addToast('success', 'Connection Updated', `Factor linked to "${cg.name || 'condition group'}" successfully`);
        emit('trigger-event', { name: 'connection-changed', event: { factorId, conditionGroupId: cg.id, action: 'linked' } });
      } catch (e) {
        errMsg('Link failed', e);
        addToast('error', 'Link Failed', e?.message || 'Failed to link factor to condition group');
      }
      scheduleLineUpdate();
    }

    onMounted(() => { if (props.content?.authToken) loadAll(); ro = new ResizeObserver(scheduleLineUpdate); nextTick(() => { if (layoutRef.value) ro.observe(layoutRef.value); }); });
    onBeforeUnmount(() => { ro?.disconnect(); clearTimeout(lineTimer); });
    watch(() => props.content?.authToken, t => { if (t) loadAll(); });

    return {
      rootRef, layoutRef, svgRef, content: computed(() => props.content),
      factorGroups, allCondGroups, entityOptions, ticketTypes,
      loadingFactorGroups, loadingConditionGroups, loadError, expandedConds,
      searchFactorGroup, searchCondGroup, filterFactorType,
      createMenuOpen, toasts,
      panel, editingFactor, editingGroupId, editingCondGroup, editingFactorGroupData,
      showModal, modalType, hoveredLine, connectPopup,
      linkedGroupEntries, unlinkedGroupsWithFactors, emptyGroups,
      hasUnlinkedSection, unlinkedCondGroups, lines,
      getGroupColor, getGroupName, cgIconStyle, sidebarStyle,
      getFactorTitle, getFactorType, formatEntity, formatThreshold,
      isProductEntityType,
      toggleCondExpand, addToast, removeToast,
      handleAddFactor, handleEditFactor, handleEditFactorGroup,
      handleEditConditionGroup, handleConnectFactor,
      openCreateFactorGroup, openCreateConditionGroup, handleModalSave,
      saveFactorConfig, saveCondGroupConfig, handleConnectSelect,
      handleDeleteFactor, handleDeleteFactorGroup, handleDeleteCondGroup,
      scheduleLineUpdate, retryLoad, refreshData: loadAll, closePanel: () => { panel.value = null; },
    };
  },
};
</script>

<style scoped lang="scss">
@import 'polaris-weweb-styles';

$card-height: 60px;
$sidebar-width: 160px;
$left-width: 560px;
$right-width: 520px;

.es {
  @include polaris-tokens;
  font-family: var(--p-font-family-sans);
  color: var(--p-color-text);
  background: var(--p-color-bg-surface-secondary);
  width: 100%; height: 100%;
  position: relative;
  overflow: hidden;

  // ═══ TOASTS ═══
  &__toasts {
    position: absolute;
    top: var(--p-space-300);
    right: var(--p-space-400);
    z-index: var(--p-z-index-5);
    display: flex;
    flex-direction: column;
    gap: var(--p-space-200);
    max-width: 380px;
    pointer-events: none;
  }
  &__toast {
    pointer-events: auto;
    border-radius: var(--p-border-radius-200);
    box-shadow: var(--p-shadow-400);
    overflow: hidden;
    animation: es-toast-in 0.25s ease-out;

    &--success {
      .es__toast-header {
        background: var(--p-color-bg-fill-inverse);
        color: var(--p-color-text-on-color);
      }
      .es__toast-body {
        background: var(--p-color-bg-surface);
        color: var(--p-color-text);
      }
    }
    &--error {
      .es__toast-header {
        background: var(--p-color-bg-fill-critical);
        color: var(--p-color-text-on-color);
      }
      .es__toast-body {
        background: var(--p-color-bg-surface);
        color: var(--p-color-text);
      }
    }
  }
  &__toast-header {
    display: flex;
    align-items: center;
    gap: var(--p-space-200);
    padding: var(--p-space-250) var(--p-space-300);
    font-size: var(--p-font-size-300);
    font-weight: 600;
  }
  &__toast-title { flex: 1; }
  &__toast-close {
    width: var(--p-space-500); height: var(--p-space-500);
    display: flex; align-items: center; justify-content: center;
    background: none; border: none; color: inherit; cursor: pointer;
    opacity: 0.7; border-radius: var(--p-border-radius-100);
    &:hover { opacity: 1; background: var(--p-color-bg-fill-transparent-hover); }
  }
  &__toast-body {
    padding: var(--p-space-250) var(--p-space-300);
    font-size: var(--p-font-size-300);
    line-height: 1.4;
    border-top: 1px solid var(--p-color-border);
  }

  @keyframes es-toast-in {
    from { opacity: 0; transform: translateX(var(--p-space-500)); }
    to { opacity: 1; transform: translateX(0); }
  }

  // ═══ ERROR BANNER ═══
  &__error-banner {
    @include polaris-banner-critical;
    margin: var(--p-space-400) var(--p-space-600) 0;
    display: flex;
    align-items: flex-start;
    gap: var(--p-space-200);
    padding: var(--p-space-300) var(--p-space-400);
    border-radius: var(--p-border-radius-200);
    background: var(--p-color-bg-surface-critical);
    border: 1px solid var(--p-color-border-critical);
    color: var(--p-color-text-critical);
    font-size: var(--p-font-size-300);
    line-height: 1.4;
    > svg { flex-shrink: 0; margin-top: var(--p-space-050); }
  }
  &__error-banner-content {
    flex: 1; min-width: 0;
    strong { display: block; font-weight: var(--p-font-weight-semibold); margin-bottom: var(--p-space-050); }
    span { color: var(--p-color-text-secondary); }
  }
  &__error-banner-action {
    @include polaris-button-plain;
    font-size: var(--p-font-size-300);
    font-weight: var(--p-font-weight-semibold);
    color: var(--p-color-text-critical);
    white-space: nowrap;
    flex-shrink: 0;
    &:hover { text-decoration: underline; }
  }
  &__error-banner-dismiss {
    width: var(--p-space-600); height: var(--p-space-600); min-width: var(--p-space-600);
    display: flex; align-items: center; justify-content: center;
    background: none; border: none; border-radius: var(--p-border-radius-100);
    color: var(--p-color-text-critical); cursor: pointer; flex-shrink: 0;
    opacity: 0.7;
    &:hover { opacity: 1; background: var(--p-color-bg-surface-critical-hover); }
  }

  &__scroll-area {
    width: 100%; height: 100%;
    overflow: auto;
    padding: 0 var(--p-space-600) 64px;
    position: relative;
  }

  &__page-header {
    display: flex; align-items: flex-start; justify-content: space-between;
    padding: var(--p-space-500) 0 var(--p-space-500);
    gap: var(--p-space-600);
    margin-bottom: var(--p-space-400);
  }
  &__page-header-left { flex-shrink: 0; }
  &__page-title {
    font-size: var(--p-font-size-400); font-weight: var(--p-font-weight-semibold);
    color: var(--p-color-text); margin: 0 0 var(--p-space-100); line-height: 1.3;
  }
  &__page-desc {
    font-size: var(--p-font-size-300); color: var(--p-color-text-secondary);
    margin: 0; line-height: 1.4;
  }
  &__page-header-right { display: flex; align-items: center; flex-shrink: 0; }
  &__filter-bar {
    display: flex; align-items: center; gap: var(--p-space-200);
  }
  &__search-input {
    width: 220px; min-height: var(--p-space-800);
    padding: var(--p-space-150) var(--p-space-300) var(--p-space-150) 36px;
    font-family: var(--p-font-family-sans); font-size: var(--p-font-size-300);
    color: var(--p-color-text); background: var(--p-color-bg-surface);
    border: 1px solid var(--p-color-border); border-radius: var(--p-border-radius-200);
    background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 20 20'%3E%3Cpath fill='%235c5f62' d='M8 12a4 4 0 1 1 0-8 4 4 0 0 1 0 8m9.707 4.293-4.82-4.82A5.968 5.968 0 0 0 14 8 6 6 0 0 0 2 8a6 6 0 0 0 6 6 5.968 5.968 0 0 0 3.473-1.113l4.82 4.82a.997.997 0 0 0 1.414 0 .999.999 0 0 0 0-1.414'/%3E%3C/svg%3E");
    background-repeat: no-repeat; background-position: 10px center; background-size: 16px;
    &::placeholder { color: var(--p-color-text-secondary); }
    &:focus { outline: none; border-color: var(--p-color-focus-ring); box-shadow: 0 0 0 1px var(--p-color-focus-ring); }
  }
  &__filter-select {
    min-height: var(--p-space-800); padding: var(--p-space-150) 28px var(--p-space-150) var(--p-space-250);
    font-family: var(--p-font-family-sans); font-size: var(--p-font-size-300);
    color: var(--p-color-text); background: var(--p-color-bg-surface);
    border: 1px solid var(--p-color-border); border-radius: var(--p-border-radius-200);
    appearance: none; cursor: pointer;
    background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 20 20'%3E%3Cpath fill='%235c5f62' d='M10 14a.997.997 0 01-.707-.293l-5-5a.999.999 0 111.414-1.414L10 11.586l4.293-4.293a.999.999 0 111.414 1.414l-5 5A.997.997 0 0110 14z'/%3E%3C/svg%3E");
    background-repeat: no-repeat; background-position: right var(--p-space-150) center; background-size: var(--p-space-400);
    &:focus { outline: none; border-color: var(--p-color-focus-ring); box-shadow: 0 0 0 1px var(--p-color-focus-ring); }
  }

  &__layout { display: flex; flex-direction: column; position: relative; }
  &__header-row { display: flex; justify-content: space-between; padding-bottom: var(--p-space-300); }
  &__col-label {
    font-size: var(--p-font-size-275); font-weight: 600; letter-spacing: 0.8px;
    text-transform: uppercase; color: var(--p-color-text-secondary);
    &--left { width: $left-width; flex-shrink: 0; }
    &--right { width: $right-width; flex-shrink: 0; }
  }
  &__primary-btn { @include polaris-button-primary; @include polaris-button-slim; font-size: var(--p-font-size-300); gap: var(--p-space-100); }

  &__create-dropdown {
    position: relative;
  }
  &__create-menu {
    position: absolute; top: 100%; right: 0; margin-top: 0;
    flex-direction: column; min-width: 210px; padding: var(--p-space-150) 0;
    background: var(--p-color-bg-surface); border: 1px solid var(--p-color-border);
    border-radius: var(--p-border-radius-200); box-shadow: var(--p-shadow-popover);
    z-index: var(--p-z-index-2);
    display: flex;

    &::before {
      content: '';
      position: absolute;
      top: -6px;
      left: 0; right: 0;
      height: var(--p-space-150);
    }
  }
  &__create-menu-item {
    display: flex; align-items: center; gap: var(--p-space-200);
    padding: var(--p-space-250) var(--p-space-350); width: 100%;
    font-family: var(--p-font-family-sans); font-size: var(--p-font-size-300);
    color: var(--p-color-text); background: none; border: none;
    cursor: pointer; text-align: left;
    transition: background 0.1s;
    &:hover { background: var(--p-color-bg-surface-hover); }
    svg { color: var(--p-color-icon); flex-shrink: 0; }
  }

  &__group-row {
    display: flex; justify-content: space-between; align-items: flex-start;
    margin-bottom: var(--p-space-300);
  }

  &__group-left {
    width: $left-width; flex-shrink: 0; z-index: 2;
    display: flex; align-items: stretch; gap: var(--p-space-200);
  }

  &__group-sidebar {
    width: $sidebar-width; flex-shrink: 0;
    min-height: $card-height;
    display: flex; align-items: stretch;
    background: var(--p-color-bg-surface);
    border: 1px solid var(--p-color-border-info);
    border-radius: var(--p-border-radius-200);
    box-shadow: var(--p-shadow-card-sm);
    overflow: hidden;
    transition: box-shadow 0.15s, border-width 0.15s;
    &:hover {
      border-width: 1.5px;
      box-shadow: var(--p-shadow-200);
      .es__sidebar-action { opacity: 0.7; }
    }
  }
  &__sidebar-accent { width: var(--p-space-100); min-width: var(--p-space-100); }
  &__sidebar-inner {
    flex: 1; padding: 0 var(--p-space-150) 0 var(--p-space-250);
    display: flex; align-items: center; gap: var(--p-space-100);
    align-self: flex-start; min-height: $card-height;
  }
  &__sidebar-name {
    flex: 1; min-width: 0;
    font-size: var(--p-font-size-275); line-height: 1.35;
    font-weight: var(--p-font-weight-semibold);
    color: var(--p-color-text);
    word-break: break-word;
  }
  &__sidebar-action {
    width: 22px; height: 22px; min-width: 22px;
    display: flex; align-items: center; justify-content: center;
    background: none; border: none; border-radius: var(--p-border-radius-100);
    color: var(--p-color-icon); cursor: pointer; flex-shrink: 0;
    opacity: 0; transition: opacity 0.15s, background 0.1s;
    &:hover { opacity: 1 !important; background: var(--p-color-bg-fill-transparent-hover); }
  }

  &__group-cards {
    flex: 1; display: flex; flex-direction: column;
    gap: var(--p-space-200); min-width: 0;
  }

  &__group-right { width: $right-width; flex-shrink: 0; z-index: 2; }
  &__cond-slots { display: flex; flex-direction: column; gap: var(--p-space-200); }
  &__cond-slot { min-height: $card-height; display: flex; align-items: flex-start; }

  &__svg {
    position: absolute; top: 0; left: 0; width: 100%; height: 100%;
    z-index: 1; pointer-events: none; overflow: visible;
    path { pointer-events: stroke; }
  }

  // ═══ UNLINKED ═══
  &__unlinked-row { display: flex; justify-content: space-between; align-items: flex-start; }
  &__unlinked-left { width: $left-width; flex-shrink: 0; display: flex; flex-direction: column; gap: var(--p-space-200); }
  &__unlinked-group {
    display: flex; align-items: stretch; gap: var(--p-space-200);
    opacity: 0.7; transition: opacity 0.15s;
    &:hover { opacity: 1; }
    &--empty { opacity: 0.5; }
  }
  &__unlinked-right { width: $right-width; flex-shrink: 0; display: flex; flex-direction: column; gap: var(--p-space-200); }

  // ─── Card ───
  &__card {
    position: relative; display: flex; align-items: stretch;
    height: $card-height;
    background: var(--p-color-bg-surface);
    border: 1px solid var(--p-color-border-info);
    border-radius: var(--p-border-radius-200);
    box-shadow: var(--p-shadow-card-sm);
    overflow: visible; transition: box-shadow 0.15s, border-color 0.15s, border-width 0.15s;
    &:hover { border-width: 1.5px; box-shadow: var(--p-shadow-200); .es__card-edit { opacity: 1; } .es__card-connect { opacity: 1; } }
    &--factor { }
    &--condition { padding-left: var(--p-space-300); width: 100%; cursor: pointer; }
    &--dim { opacity: 0.6; border-color: var(--p-color-border); box-shadow: none; &:hover { opacity: 1; border-color: var(--p-color-border-info); } }
    &--expanded { height: auto; }
  }
  &__card-accent { width: var(--p-space-100); min-width: var(--p-space-100); border-radius: var(--p-border-radius-200) 0 0 var(--p-border-radius-200); }
  &__card-dot-indicator {
    position: absolute; left: -7px; top: 30px; transform: translateY(-50%);
    width: 10px; height: 10px; border-radius: 50%;
    background: var(--p-color-border-info); border: 2px solid var(--p-color-bg-surface-secondary); z-index: 3;
  }
  &__card-body { flex: 1; display: flex; align-items: center; gap: var(--p-space-200); padding: 0 var(--p-space-300); min-width: 0; }
  &__card-icon {
    width: 28px; height: 28px; min-width: 28px; border-radius: var(--p-border-radius-100);
    border: 0.7px solid var(--p-color-border); display: flex; align-items: center; justify-content: center;
    &--points { color: var(--p-color-text-info); }
    &--credit { color: var(--p-color-text); }
  }
  &__cg-icon { width: 28px; height: 28px; min-width: 28px; border-radius: var(--p-border-radius-100); display: flex; align-items: center; justify-content: center; }
  &__card-info { flex: 1; min-width: 0; }
  &__card-name { font-size: var(--p-font-size-325); font-weight: var(--p-font-weight-semibold); color: var(--p-color-text); white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
  &__card-sub { font-size: var(--p-font-size-275); color: var(--p-color-text-secondary); display: flex; align-items: center; gap: var(--p-space-150); white-space: nowrap; }
  &__card-dot-sep { width: 3px; height: 3px; border-radius: 50%; background: var(--p-color-text-disabled); flex-shrink: 0; }

  &__card-badge {
    font-size: var(--p-font-size-325); font-weight: var(--p-font-weight-bold);
    white-space: nowrap; flex-shrink: 0;
    &--mult { color: var(--p-color-text-success); }
    &--rate { color: var(--p-color-text-info); }
  }

  &__card-edit {
    width: var(--p-space-600); height: var(--p-space-600); display: flex; align-items: center; justify-content: center;
    background: none; border: none; border-radius: var(--p-border-radius-100);
    color: var(--p-color-icon); cursor: pointer; opacity: 0; transition: opacity 0.1s; flex-shrink: 0;
    &:hover { background: var(--p-color-bg-fill-transparent-hover); }
    &--visible { opacity: 1; }
  }
  &__card-connect {
    position: absolute; right: -15px; top: 50%; transform: translateY(-50%);
    width: 28px; height: 28px; border-radius: 50%;
    background: var(--p-color-bg-fill-brand); color: var(--p-color-text-on-color);
    border: 3px solid var(--p-color-bg-surface-secondary); display: flex; align-items: center; justify-content: center;
    cursor: pointer; opacity: 0; z-index: 5; box-shadow: var(--p-shadow-button);
    transition: opacity 0.1s, background 0.1s, transform 0.1s;
    &:hover { background: var(--p-color-bg-fill-brand-hover); transform: translateY(-50%) scale(1.1); }
  }
  &__link-btn { @include polaris-button-plain; font-size: var(--p-font-size-275); white-space: nowrap; padding: var(--p-space-050) var(--p-space-150); min-height: auto; }

  // ─── Linked badge ───
  &__linked-badge {
    display: inline-flex; align-items: center; gap: var(--p-space-100);
    padding: var(--p-space-050) var(--p-space-200) var(--p-space-050) 5px; border-radius: var(--p-border-radius-300);
    background: var(--p-color-bg-fill-info-secondary); color: var(--p-color-text-info);
    font-size: var(--p-font-size-275); font-weight: var(--p-font-weight-semibold);
    svg { flex-shrink: 0; }
  }

  // ─── Condition expand ───
  &__cond-wrap { flex: 1; display: flex; flex-direction: column; min-width: 0; }
  &__cond-header {
    height: 58px; display: flex; align-items: center;
    gap: var(--p-space-200); padding: 0 var(--p-space-300);
    min-width: 0; cursor: pointer;
  }
  &__expand-btn {
    width: 28px; height: 28px; min-width: 28px;
    display: flex; align-items: center; justify-content: center;
    background: var(--p-color-bg-surface); border: 1px solid var(--p-color-border);
    border-radius: var(--p-border-radius-200); cursor: pointer; flex-shrink: 0;
    color: var(--p-color-icon); transition: background 0.1s;
    svg { transform: rotate(180deg); transition: transform 0.2s; }
    &--open svg { transform: rotate(0deg); }
    &:hover { background: var(--p-color-bg-fill-transparent-hover); }
  }

  &__cond-edit {
    width: 28px; height: 28px; min-width: 28px;
    display: flex; align-items: center; justify-content: center;
    background: none; border: none; border-radius: var(--p-border-radius-100);
    color: var(--p-color-icon); cursor: pointer; flex-shrink: 0;
    opacity: 0; transition: opacity 0.15s, background 0.1s;
    &:hover { background: var(--p-color-bg-fill-transparent-hover); opacity: 1; }
  }
  &__card--condition:hover &__cond-edit { opacity: 0.7; }

  &__cond-detail { padding: 0 var(--p-space-300) var(--p-space-300); }

  &__cond-table-wrap {
    border: 1px solid var(--p-color-border);
    border-radius: var(--p-border-radius-200);
    overflow: hidden;
  }

  &__col-type { width: 91px; }
  &__col-items { width: 64px; }
  &__col-logic { width: 70px; }
  &__col-threshold { width: 165px; }
  &__col-excess { }

  &__cond-table {
    width: 100%; border-collapse: collapse; font-size: var(--p-font-size-275);
    font-family: var(--p-font-family-sans);
    table-layout: fixed;
    th {
      text-align: center; padding: var(--p-space-100) 0;
      font-weight: 400;
      color: var(--p-color-text); font-size: var(--p-font-size-275);
      background: var(--p-color-bg-surface-secondary);
      border-bottom: 1px solid var(--p-color-border);
      white-space: nowrap; overflow: hidden; text-overflow: ellipsis;
      &:first-child { border-left: none; }
      &:last-child { border-right: none; }
    }
    th + th { border-left: 1px solid var(--p-color-border); }
    td {
      text-align: center; padding: var(--p-space-150) 0;
      border-bottom: 1px solid var(--p-color-border);
      color: var(--p-color-text); font-size: var(--p-font-size-275);
      vertical-align: middle;
      white-space: nowrap; overflow: hidden; text-overflow: ellipsis;
    }
    td + td { border-left: 1px solid var(--p-color-border); }
    tr:last-child td { border-bottom: none; }
  }

  &__items-badge {
    display: inline-flex; align-items: center; gap: var(--p-space-100);
    padding: var(--p-space-050) var(--p-space-200); border-radius: var(--p-border-radius-200);
    background: var(--p-color-bg-fill-magic-secondary); color: var(--p-color-text-magic);
    font-size: var(--p-font-size-275); font-weight: 500;
    height: var(--p-space-500); box-sizing: border-box;
    line-height: 1;
    svg { opacity: 0.64; flex-shrink: 0; }
  }

  &__threshold-cell {
    display: inline-flex; align-items: center; justify-content: center; gap: var(--p-space-100);
    width: 100%; padding: 0 var(--p-space-300); box-sizing: border-box;
    span { white-space: nowrap; }
    svg { color: var(--p-color-icon-secondary); flex-shrink: 0; }
  }

  // ─── Divider ───
  &__divider {
    display: flex; align-items: center; gap: var(--p-space-300);
    padding: var(--p-space-400) 0;
    span { font-size: 10px; /* TODO: verify Polaris token — no standard token for 10px */ font-weight: var(--p-font-weight-semibold); color: var(--p-color-text-disabled); text-transform: uppercase; letter-spacing: 1px; }
    &::after { content: ''; flex: 1; height: var(--p-space-050); background: var(--p-color-bg-surface-tertiary); border-radius: 1px; }
  }

  &__loading { display: flex; justify-content: center; padding: var(--p-space-1200) 0; }
  &__spinner { @include polaris-spinner; }
  &__empty {
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    min-height: 280px;
    padding: var(--p-space-800) var(--p-space-400);
    background: var(--p-color-bg-surface);
    border: 1px solid var(--p-color-border);
    border-radius: var(--p-border-radius-300);
    text-align: center;
  }
  &__empty-icon {
    width: 48px;
    height: 48px;
    border-radius: var(--p-border-radius-200);
    background: var(--p-color-bg-surface-secondary);
    display: flex;
    align-items: center;
    justify-content: center;
    font-size: var(--p-font-size-750, 24px);
    margin-bottom: var(--p-space-400);
  }
  &__empty-heading {
    font-size: var(--p-font-size-400);
    font-weight: var(--p-font-weight-semibold);
    color: var(--p-color-text);
    margin: 0 0 var(--p-space-200);
  }
  &__empty-desc {
    font-size: var(--p-font-size-300);
    color: var(--p-color-text-secondary);
    margin: 0 0 var(--p-space-400);
    max-width: 360px;
    line-height: 1.5;
  }
  &__empty-action {
    @include polaris-button-primary;
    @include polaris-button-slim;
    font-size: var(--p-font-size-300);
    gap: var(--p-space-100);
  }

  &__panel-backdrop {
    position: absolute;
    top: 0; left: 0; width: 100%; height: 100%;
    background: var(--p-color-bg-backdrop, rgba(0, 0, 0, 0.3));
    z-index: var(--p-z-index-3);
    cursor: pointer;
  }

  &__config-panel {
    position: absolute;
    top: 0; right: 0; width: auto; height: 100%;
    z-index: var(--p-z-index-4);
  }
}
</style>
