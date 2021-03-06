<template>
  <v-container>
    <v-form ref="createAccountInfoForm" lazy-validation>
      <p class="mb-10">
        You must be the Prime Contact to link this account with your existing BC Online account.
      </p>
      <BcolLogin @account-link-successful="onLink" v-show="!linked"></BcolLogin>
      <template v-if="linked">
        <v-alert dark color="primary" icon="mdi-check" class="py-6 pr-6 pl-4" v-model="linked">
          <div class="bcol-acc d-flex justify-space-between align-center">
            <div v-if="currentOrganization.bcolAccountDetails">
              <div class="bcol-acc__link-status mb-3">Account Linked!</div>
              <div class="bcol-acc__name">
                {{ currentOrganization.name }}
              </div>
              <ul class="bcol-acc__meta">
                <li>
                  Account No: {{ currentOrganization.bcolAccountDetails.accountNumber }}
                </li>
                <li>
                  Authorizing User ID: {{ currentOrganization.bcolAccountDetails.userId }}
                </li>
              </ul>
            </div>
            <div>
              <v-btn
                large
                outlined
                class="font-weight-bold"
                @click="unlinkAccounts()"
                data-test="dialog-save-button"
              >
                Remove Linked Accounts
              </v-btn>
            </div>
          </div>
        </v-alert>
        <v-checkbox large color="primary" v-model="grantAccess" class="bcol-auth mt-8">
          <template v-slot:label>
            <div class="bcol-auth__label" v-html="grantAccessText"></div>
          </template>
        </v-checkbox>
        <template v-if="grantAccess">
          <v-row class="mt-6">
            <v-col cols="12">
              <h3 class="mb-4">Account Information</h3>
              <p class="mb-0">
                The following information will be imported from your existing BC Online account.
              </p>
              <p class="mb-8">
                Review your account information below and update if needed.
              </p>
            </v-col>
          </v-row>
          <fieldset class="mb-2">
            <legend class="mb-3">Account Name</legend>
            <v-text-field
              filled
              label="Account Name"
              v-model.trim="currentOrganization.name"
              persistent-hint
              disabled
              data-test="account-name"
            >
            </v-text-field>
          </fieldset>
          <BaseAddress :inputAddress="address" @address-update="updateAddress">
          </BaseAddress>
        </template>
        <v-alert type="error" class="mb-6" v-show="errorMessage">
          {{ errorMessage }}
        </v-alert>
      </template>
      <v-row>
        <v-col cols="12" class="step-btns mt-8 pb-0 d-inline-flex">
          <v-btn large color="default" @click="goBack">
            <v-icon left class="mr-1">mdi-arrow-left</v-icon>
            Back
          </v-btn>
          <v-spacer></v-spacer>
          <v-btn class="mr-3" large depressed color="primary" :disabled="!grantAccess" @click="save">
            Next
            <v-icon right class="ml-1">mdi-arrow-right</v-icon>
          </v-btn>
          <ConfirmCancelButton
            :showConfirmPopup="linked"
          ></ConfirmCancelButton>
        </v-col>
      </v-row>
    </v-form>
  </v-container>
</template>

<script lang="ts">
import { BcolAccountDetails, BcolProfile } from '@/models/bcol'
import { Component, Mixins, Prop, Vue } from 'vue-property-decorator'
import { CreateRequestBody, Member, Organization } from '@/models/Organization'
import { mapActions, mapMutations, mapState } from 'vuex'
import { Account } from '@/util/constants'
import { Address } from '@/models/address'
import BaseAddress from '@/components/auth/BaseAddress.vue'
import BcolLogin from '@/components/auth/BcolLogin.vue'
import ConfirmCancelButton from '@/components/auth/common/ConfirmCancelButton.vue'
import { KCUserProfile } from 'sbc-common-components/src/models/KCUserProfile'
import OrgModule from '@/store/modules/org'
import Steppable from '@/components/auth/stepper/Steppable.vue'
import { getModule } from 'vuex-module-decorators'

@Component({
  components: {
    BcolLogin,
    BaseAddress,
    ConfirmCancelButton
  },
  computed: {
    ...mapState('org', ['currentOrganization']),
    ...mapState('user', ['userProfile', 'currentUser'])
  },
  methods: {
    ...mapMutations('org', [
      'setCurrentOrganization',
      'setCurrentOrganizationAddress',
      'setGrantAccess'
    ]),
    ...mapActions('org', [
      'createOrg',
      'syncMembership',
      'syncOrganization',
      ''
    ])
  }
})
export default class AccountCreatePremium extends Mixins(Steppable) {
  private orgStore = getModule(OrgModule, this.$store)
  private username = ''
  private password = ''
  private errorMessage: string = ''
  private saving = false
  private readonly createOrg!: () => Promise<Organization>
  private readonly syncMembership!: (orgId: number) => Promise<Member>
  private readonly syncOrganization!: (orgId: number) => Promise<Organization>
  private readonly currentOrganization!: Organization
  private readonly currentUser!: KCUserProfile
  private readonly setCurrentOrganization!: (organization: Organization) => void
  private readonly setCurrentOrganizationAddress!: (address: Address) => void
  private readonly setGrantAccess!: (grantAccess: boolean) => void

  get grantAccessText () {
    return `I ,<strong>${this.currentUser?.fullName} </strong>, confirm that I am authorized to grant access to the account <strong>${this.currentOrganization?.bcolAccountDetails?.orgName}</strong>`
  }

  get grantAccess () {
    return this.currentOrganization?.grantAccess
  }
  set grantAccess (grantAccess: boolean) {
    this.setGrantAccess(grantAccess)
  }
  $refs: {
    createAccountInfoForm: HTMLFormElement
  }

  private readonly teamNameRules = [v => !!v || 'An account name is required']

  private isFormValid (): boolean {
    return !!this.username && !!this.password
  }

  private get address () {
    return this.currentOrganization.bcolAccountDetails.address
  }
  private set address (address: Address) {
    this.setCurrentOrganizationAddress(address)
  }
  private unlinkAccounts () {
    this.setCurrentOrganization(undefined)
  }
  private get linked () {
    return !!this.currentOrganization?.bcolAccountDetails
  }
  private updateAddress (address: Address) {
    this.address = address
  }
  private async save () {
    // TODO Handle edit mode as well here
    this.goNext()
  }

  private onLink (details: {
    bcolProfile: BcolProfile
    bcolAccountDetails: BcolAccountDetails
  }) {
    var org: Organization = {
      name: details.bcolAccountDetails.orgName,
      accessType: Account.PREMIUM,
      bcolProfile: details.bcolProfile,
      bcolAccountDetails: details.bcolAccountDetails,
      grantAccess: false
    }
    this.setCurrentOrganization(org)
  }
  private cancel () {
    if (this.stepBack) {
      this.stepBack()
    } else {
      this.$router.push({ path: '/home' })
    }
  }

  private goBack () {
    this.stepBack()
  }

  private goNext () {
    this.stepForward()
  }

  private redirectToNext (organization?: Organization) {
    this.$router.push({ path: `/account/${organization.id}/` })
  }
}
</script>

<style lang="scss" scoped>
@import '$assets/scss/theme.scss';

// Tighten up some of the spacing between rows
[class^='col'] {
  padding-top: 0;
  padding-bottom: 0;
}

.form__btns {
  display: flex;
  justify-content: flex-end;
}

.bcol-acc__link-status {
  text-transform: uppercase;
  font-size: 0.9375rem;
}

.bcol-acc {
  margin-top: 2px;
  margin-bottom: 2px;
}

.bcol-acc__name {
  font-size: 1.25rem;
  font-weight: 700;
}

.bcol-acc__meta {
  margin: 0;
  padding: 0;
  list-style-type: none;

  li {
    position: relative;
    display: inline-block
  }

  li + li {
    &:before {
      content: ' | ';
      display: inline-block;
      position: relative;
      top: -2px;
      width: 2rem;
      vertical-align: top;
      text-align: center;
    }
  }
}

.bcol-auth {
  max-width: 40rem;

  ::v-deep {
    .v-input__slot {
      align-items: flex-start;
    }
  }
}

.bcol-auth__label {
  margin-left: 0.5rem;
  line-height: 1.5;
  color: var(--v-grey-darken4) !important;
}
</style>
