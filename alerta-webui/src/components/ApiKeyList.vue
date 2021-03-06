<template>
  <div>
    <v-dialog
      v-model="dialog"
      max-width="500px"
    >
      <v-form ref="form">
        <v-card>
          <v-card-title>
            <span class="headline">
              {{ formTitle }}
            </span>
          </v-card-title>

          <v-card-text>
            <v-container grid-list-md>
              <v-layout wrap>
                <v-flex
                  xs12
                >
                  <v-tooltip
                    :key="copyIconText"
                    right
                  >
                    <v-text-field
                      v-if="editedItem.key"
                      slot="activator"
                      v-model="editedItem.key"
                      label="API Key"
                      readonly
                      monospace
                      append-icon="content_copy"
                      @click:append="clipboardCopy(editedItem.key)"
                    />
                    <span>{{ copyIconText }}</span>
                  </v-tooltip>
                </v-flex>
                <v-flex
                  v-if="$config.customer_views"
                  xs12
                >
                  <v-select
                    v-model="editedItem.customer"
                    :items="allowedCustomers"
                    label="Customer"
                  />
                </v-flex>
                <v-flex
                  xs12
                >
                  <v-autocomplete
                    v-model="editedItem.scopes"
                    :items="allowedScopes"
                    label="Scopes"
                    chips
                    clearable
                    solo
                    multiple
                  >
                    <template
                      slot="selection"
                      slot-scope="data"
                    >
                      <v-chip
                        :selected="data.selected"
                        close
                      >
                        <strong>{{ data.item }}</strong>&nbsp;
                        <span>(scope)</span>
                      </v-chip>
                    </template>
                  </v-autocomplete>
                </v-flex>
                <v-flex
                  xs12
                >
                  <v-menu
                    v-model="menu"
                    :close-on-content-click="false"
                    :nudge-right="40"
                    lazy
                    transition="scale-transition"
                    offset-y
                    full-width
                    min-width="290px"
                  >
                    <v-text-field
                      slot="activator"
                      v-model="pickerDate"
                      label="Expires"
                      prepend-icon="event"
                      readonly
                    />
                    <v-date-picker
                      v-model="pickerDate"
                      :min="new Date().toISOString().slice(0, 10)"
                      @input="menu = false"
                    />
                  </v-menu>
                </v-flex>
                <v-flex
                  xs12
                >
                  <v-text-field
                    v-model="editedItem.text"
                    label="Comment"
                  />
                </v-flex>
              </v-layout>
            </v-container>
          </v-card-text>

          <v-card-actions>
            <v-spacer />
            <v-btn
              color="blue darken-1"
              flat
              @click="close"
            >
              Cancel
            </v-btn>
            <v-btn
              color="blue darken-1"
              flat
              @click="save"
            >
              Save
            </v-btn>
          </v-card-actions>
        </v-card>
      </v-form>
    </v-dialog>

    <v-card>
      <v-card-title class="title">
        API Keys
        <v-spacer />
        <v-text-field
          v-model="search"
          append-icon="search"
          label="Search"
          single-line
          hide-details
        />
      </v-card-title>

      <v-data-table
        :headers="computedHeaders"
        :items="keys"
        :rows-per-page-items="rowsPerPageItems"
        :pagination.sync="pagination"
        class="px-2"
        :search="search"
        :loading="isLoading"
        must-sort
        sort-icon="arrow_drop_down"
      >
        <template
          slot="items"
          slot-scope="props"
        >
          <td
            class="text-no-wrap"
            monospace
          >
            {{ props.item.key }}
            <v-tooltip
              :key="copyIconText"
              top
            >
              <v-icon
                slot="activator"
                :value="props.item.key"
                style="font-size: 16px;"
                @click="clipboardCopy(props.item.key)"
              >
                content_copy
              </v-icon>
              <span>{{ copyIconText }}</span>
            </v-tooltip>
          </td>
          <td>
            <v-tooltip
              v-if="!isExpired(props.item.expires)"
              top
            >
              <v-icon
                slot="activator"
                color="primary"
                small
              >
                check_circle
              </v-icon>
              <span>Active</span>
            </v-tooltip>
            <v-tooltip
              v-if="isExpired(props.item.expires)"
              top
            >
              <v-icon
                slot="activator"
                small
              >
                error_outline
              </v-icon>
              <span>Expired</span>
            </v-tooltip>
          </td>
          <td>{{ props.item.user }}</td>
          <td>
            <v-chip
              v-for="scope in props.item.scopes"
              :key="scope"
              small
            >
              <strong>{{ scope }}</strong>&nbsp;
              <span>(scope)</span>
            </v-chip>
          </td>
          <td>{{ props.item.text }}</td>
          <td>
            <date-time
              :value="props.item.expireTime"
              format="mediumDate"
            />
          </td>
          <td
            class="text-xs-center"
          >
            {{ props.item.count }}
          </td>
          <td>{{ props.item.lastUsedTime | timeago }}</td>
          <td
            v-if="$config.customer_views"
          >
            {{ props.item.customer }}
          </td>
          <td class="text-no-wrap">
            <v-btn
              v-has-perms.disable="'write:keys'"
              icon
              class="btn--plain mr-0"
              @click="editItem(props.item)"
            >
              <v-icon
                small
                color="grey darken-3"
              >
                edit
              </v-icon>
            </v-btn>
            <v-btn
              v-has-perms.disable="'admin:keys'"
              icon
              class="btn--plain mx-0"
              @click="deleteItem(props.item)"
            >
              <v-icon
                small
                color="grey darken-3"
              >
                delete
              </v-icon>
            </v-btn>
            <v-btn
              v-has-perms.disable="'admin:keys'"
              :href="`data:text/plain;base64,${toData(props.item)}`"
              :download="`key_${props.item.id}.json`"
              icon
              class="btn--plain mx-0"
            >
              <v-icon
                small
                color="grey darken-3"
              >
                get_app
              </v-icon>
            </v-btn>
          </td>
        </template>
        <template slot="no-data">
          <v-alert
            :value="true"
            color="error"
            icon="warning"
          >
            Sorry, nothing to display here :(
          </v-alert>
        </template>
        <v-alert
          slot="no-results"
          :value="true"
          color="error"
          icon="warning"
        >
          Your search for "{{ search }}" found no results.
        </v-alert>
      </v-data-table>
    </v-card>

    <list-button-add
      perms="write:keys"
      @add-to-list="dialog = true"
    />
  </div>
</template>

<script>
import DateTime from './lib/DateTime'
import ListButtonAdd from './lib/ListButtonAdd'
import utils from '@/common/utils'
import moment from 'moment'

export default {
  components: {
    DateTime,
    ListButtonAdd
  },
  data: vm => ({
    descending: true,
    page: 1,
    rowsPerPageItems: [10, 20, 30, 40, 50],
    pagination: {
      sortBy: 'lastUsedTime',
      rowsPerPage: 20
    },
    search: '',
    dialog: false,
    headers: [
      { text: 'API Key', value: 'key', sortable: false },
      { text: '', value: 'expireTime' },
      { text: 'User', value: 'user' },
      { text: 'Scopes', value: 'scopes' },
      { text: 'Description', value: 'text' },
      { text: 'Expires', value: 'expireTime' },
      { text: 'Count', value: 'count' },
      { text: 'Last Used', value: 'lastUsedTime' },
      { text: 'Customer', value: 'customer' },
      { text: 'Actions', value: 'name', sortable: false }
    ],
    editedId: null,
    editedItem: {
      key: '',
      user: '',
      text: '',
      customer: null,
      scopes: [],
      expireTime: null
    },
    menu: false,
    pickerDate: vm.defaultExpireTime(),
    defaultItem: {
      user: '',
      text: '',
      customer: null,
      scopes: [],
      expireTime: null
    },
    copyIconText: 'Copy'
  }),
  computed: {
    keys() {
      return this.$store.state.keys.keys
    },
    computedHeaders() {
      return this.headers.filter(h => !this.$config.customer_views ? h.value != 'customer' : true)
    },
    allowedScopes() {
      return utils.getAllowedScopes(
        this.$store.getters['auth/scopes'],
        this.$store.state.perms.scopes
      )
    },
    allowedCustomers() {
      return this.$store.getters['customers/customers']
    },
    isLoading() {
      return this.$store.state.keys.isLoading
    },
    formTitle() {
      return !this.editedId ? 'New API Key' : 'Edit API Key'
    },
    refresh() {
      return this.$store.state.refresh
    }
  },
  watch: {
    dialog(val) {
      val || this.close()
    },
    refresh(val) {
      val || this.getApiKeys()
    }
  },
  created() {
    this.getApiKeys()
    this.getScopes()
    this.getCustomers()
  },
  methods: {
    getApiKeys() {
      this.$store.dispatch('keys/getKeys')
    },
    getScopes() {
      this.$store.dispatch('perms/getScopes')
    },
    getCustomers() {
      this.$store.dispatch('customers/getCustomers')
    },
    defaultExpireTime() {
      return moment().add(1, 'Year').endOf('day').toISOString().slice(0, 10)
    },
    endOfDay(date) {
      let endOfDay = new Date(date)
      endOfDay.setHours(23, 59, 59, 999)
      return endOfDay.toISOString()
    },
    editItem(item) {
      this.editedId = item.id
      this.editedItem = Object.assign({}, item)
      this.pickerDate = item.expireTime.slice(0, 10)
      this.dialog = true
    },
    deleteItem(item) {
      confirm('Are you sure you want to delete this item?') &&
        this.$store.dispatch('keys/deleteKey', item.id)
    },
    close() {
      this.dialog = false
      setTimeout(() => {
        this.editedItem = Object.assign({}, this.defaultItem)
        this.pickerDate = this.defaultExpireTime()
        this.editedId = null
      }, 300)
    },
    save() {
      if (this.editedId) {
        this.$store.dispatch('keys/updateKey', [
          this.editedId,
          {
            scopes: this.editedItem.scopes,
            text: this.editedItem.text,
            expireTime: this.endOfDay(this.pickerDate),
            customer: this.editedItem.customer
          }
        ])
      } else {
        this.$store.dispatch(
          'keys/createKey',
          Object.assign(this.editedItem, {
            expireTime: this.endOfDay(this.pickerDate)
          })
        )
      }
      this.close()
    },
    isExpired(date) {
      return new Date().getTime() > new Date(date).getTime()
    },
    clipboardCopy(text) {
      this.copyIconText = 'Copied!'
      let textarea = document.createElement('textarea')
      textarea.textContent = text
      document.body.appendChild(textarea)
      textarea.select()
      document.execCommand('copy')
      document.body.removeChild(textarea)
      setTimeout(() => {
        this.copyIconText = 'Copy'
      }, 2000)
    },
    toData(item) {
      return btoa(JSON.stringify(item))
    }
  }
}
</script>

<style>
input[monospace],
td[monospace] {
  font-family: SFMono-Regular, Consolas, Liberation Mono, Menlo, Courier,
    monospace;
}
</style>
