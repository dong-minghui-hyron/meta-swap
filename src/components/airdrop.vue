<template>
  <div class="selectionWrap">
    <div class="top">
      <span class="source">{{ $t("BSV") }}</span>
      <span class="account"
        >{{ $t("account") }}：{{
          token1 === "BSV" ? fromAccount.bsv : toAccount.ftBalance
        }}</span
      >
    </div>
	<div class="top">
	  <span class="source">{{ $t("MC") }}</span>
	  <span class="account"
	    >{{ $t("account") }}：{{
	      token2 === "BSV" ? fromAccount.bsv : toAccount.ftBalance
	    }}</span
	  >
	</div>
    <div class="foot">
      <van-field
        type="number"
        :placeholder="inputPlace"
        @keyup="
          appMetaIdJsV2
            ? appCurrentQuo(payToRecevice)
            : currentQuo(payToRecevice)
        "
      />
    </div>
	<div class="foot">
	  <van-field
	    type="number"
	    :placeholder="targetAdress"
	    @keyup="
	      appMetaIdJsV2
	        ? appCurrentQuo(payToRecevice)
	        : currentQuo(payToRecevice)
	    "
	  />
	</div>
    <!--底部-->
    <div class="PayDetailWrap">
      <van-button
        type="info"
        loading-type="spinner"
        :loading="swapLoading"
        @click.stop="appMetaIdJsV2 ? appToDetail() : toDetail()"
        >{{ $t("send") }}</van-button
      >
    </div>
  </div>
</template>

<script>
import { mapState } from "vuex";
import { imgUrl, cutLength, toDecimal } from "@filters";
export default {
  name: "Selection",
  components: {},
  data() {
    return {
      token2: "MC",
      token1: "BSV",
      dialogShow: false,
      bsvDisable: false,
      ftDisable: false,
      isConfirm: false,
      newCoinList: [],
      ftList: [],
      swapLoading: false,
      coin: 0,
      swapRate: {},
      initRate: {},
      swapTxid: "",
      initAcount: 0,
      priceFloat: 0,
      oneFtRate: 0,
      oneBsvRate: 0,
      appMetaIdJsV2: {},
      initRateDate: {},
    };
  },
  mounted() {},
  computed: {
    ...mapState([
      "fromAccount",
      "toAccount",
      "userData",
      "PhoneShow",
      "appAccessToken",
    ]),
    // ...mapGetters(['isLogined']),
    inputPlace() {
      return `${this.$t("input")}`;
    },
	targetAdress() {
	  return `${this.$t("inputAddress")}`;
	},
    appInit() {
      if (this.userData && this.PhoneShow) {
        return true;
      } else return false;
    },
    //BSV转聪
    bsvToSatoshis() {
      if (this.token1 === "BSV") {
        return (this.from * Math.pow(10, 9)) / 10;
      } else {
        return this.from;
      }
    },

    initCoin() {
      return this.coin;
    },
    minRecevice() {
      if (this.token1 === "BSV" && this.recevice > 0) {
        return this.recevice;
      } else {
        return this.recevice;
      }
    },
    fee() {
      if (this.initRateDate.txFee) {
        return +this.initRateDate.txFee / Math.pow(10, 8);
      } else {
        return 0;
      }
    },
    to() {
      return this.from;
    },
    accessToken() {
      return this.$cookies.get("access_token") || "";
    },
  },
  filters: {
    imgUrl,
    cutLength,
    toDecimal,
    cutZero(val) {
      return parseFloat(val);
    },
    limited(num) {
      if (num < 0) {
        return 0.01;
      } else if (num > 100) {
        return 100;
      } else {
        return num;
      }
    },
  },
  async activated() {
    if (this.userData && !window.appMetaIdJsV2) {
      await this.getBsvAccount();
      await this.getFtList();
      await this.initSwapRate();
      this.swapTxid = "";
      this.from = null;
      this.recevice = 0;
    } else {
      return;
    }
  },
  watch: {
    $route: {
      handler(to, from) {
        if (window.appMetaIdJsV2 && from && from.name === "Detail") {
          this.appGetBsvAccount();
          this.appGetFtList();
          this.appInitSwapRate();
          this.swapTxid = "";
          this.from = null;
          this.recevice = 0;
        }
      },
    },
    dialogShow: {
      handler(newVal) {
        if (newVal === false) {
          (this.bsvDisable = false), (this.ftDisable = false);
        }
      },
    },
    userData: {
      async handler(newVal) {
        if (newVal && !window.appMetaIdJsV2) {
          await this.getBsvAccount();
          await this.getFtList();
          await this.initSwapRate();
        }
      },
    },
    "$store.state.appAccessToken": {
      handler(newVal) {
        if (newVal !== "" && window.appMetaIdJsV2) {
          this.appGetBsvAccount();
          this.appGetFtList();
          this.appInitSwapRate();
        }
      },
    },
  },
  methods: {
    //输入校验
    validator(value) {
      let decimal = +this.toAccount.ftDecimalNum;
      if (this.token1 === "BSV") {
        return value.match(/\d+.?\d{0,8}/);
      } else {
        return value.match(new RegExp(`\\d+.?\\d{0,${decimal}}`));
      }
    },

    async appInitSwapRate() {
      this.appGetInitSwapRate();
    },
    getInitSwapRate() {
      const symbol = "bsv-mc";
      return new Promise((resolve, reject) => {
        this.$store.state.metaidjs.swapreqswapargs({
          accessToken: this.accessToken,
          data: {
            symbol,
            
            op_code: this.token1 === "BSV" ? 3 : 4, //3代表Token1为bsv,Token2为Mc,BSV兑MC; 4代表Token1为bsv,Token2为Mc，MC兑BSV
          },
          callback: (res) => {
            if (res.data.code == 0) {
              this.initRate = res.data.data;
              resolve(this.initRate);
            } else {
              reject();
            }
          },
        });
      });
    },

    //交易并跳转
    async toDetail() {
      if (!this.accessToken) {
        this.$toast.fail({
          message: `${this.$t("loginStatusLost")}`,
          forbidClick: true,
          duration: 3000,
        });
        this.swapLoading = false;
        return;
      }
      await this.initSwapRate();
      const {
        ftCodehash,
        ftGenesis,
        ftSymbol,
        ftSensibleId,
        ftBalance,
        ftDecimalNum,
      } = this.toAccount;

      if (!this.from || !this.recevice) {
        this.$toast.fail({
          message: `${this.$t("unBlank")}`,
          forbidClick: true,
          duration: 3000,
        });
        this.swapLoading = false;
        return;
      }
      if (+this.from > +ftBalance && this.token1 !== "BSV") {
        this.$toast.fail({
          message: `${this.$t("mcAmountEnough")}`,
          forbidClick: true,
          duration: 3000,
        });
        this.swapLoading = false;
        return;
      }
      if (+this.from > +this.fromAccount.bsv && this.token1 === "BSV") {
        this.$toast.fail({
          message: `${this.$t("bsvAmountEnough")}`,
          forbidClick: true,
          duration: 3000,
        });
        this.swapLoading = false;
        return;
      }
      try {
        const swap2symbol = ftSymbol.toLowerCase();
        const symbol = `bsv-mc`;
        const bsvOption = {
          amount:
            this.token1 === "BSV"
              ? +this.bsvToSatoshis
              : this.recevice * Math.pow(10, 8),
          codehash: "",
          genesisId: "",
        };
        const ftOption = {
          amount:
            this.token2 === "BSV"
              ? this.from * 10 ** +this.toAccount.ftDecimalNum
              : this.recevice * Math.pow(10, +this.toAccount.ftDecimalNum),
          codehash: ftCodehash,
          genesisId: ftGenesis,
          sensibleId: ftSensibleId,
        };

        this.swapLoading = true;
        //计算转账所需手续费，若不够就不进行交易
        if (this.token1 === "BSV" && this.initRateDate.op == 3) {
          let swapFee =
            bsvOption.amount +
            this.initRateDate.txFee +
            (bsvOption.amount * this.initRateDate.swapFeeRate) / 10000;
          if (swapFee > +this.fromAccount.bsv * 10 ** 8) {
            this.$toast.fail({
              message: this.$t("changeFeeNotEnought"),
              forbidClick: true,
              duration: 3000,
            });
            this.swapLoading = false;
            return;
          }
        } else {
          let swapFee = this.initRateDate.txFee;
          if (swapFee > +this.fromAccount.bsv * 10 ** 8) {
            this.$toast.fail({
              message: this.$t("changeFeeNotEnought"),
              forbidClick: true,
              duration: 3000,
            });
            this.swapLoading = false;
            return;
          }
        }

        this.$store.state.metaidjs.swapft({
          accessToken: this.accessToken,
          data: {
            symbol,
            op_code: this.token1 === "BSV" ? 3 : 4,
            token1: bsvOption,
            token2: ftOption,
            reqswapargs: this.initRateDate,
          },

          callback: (res) => {
            if (res.code == 200) {
              if (res.data?.code == 0) {
                this.swapLoading = false;
                this.swapTxid = res.data.swapTxid;
                this.toDetailPage();
              }
            } else {
              this.$toast.fail({
                message:
                  res.data.message.indexOf("swapft failed") !== -1
                    ? `${this.$t("bsvEnough")}`
                    : `${res.data.message}`,
                forbidClick: true,
                duration: 3000,
              });

              this.swapLoading = false;
            }
          },
        });
      } catch (error) {
        this.swapLoading = false;
        this.$toast.fail({
          message: this.$t("swapFails"),
          forbidClick: true,
          duration: 3000,
        });
        throw new Error(error);
      }
    },
    //app交易并跳转成功页面
    async appToDetail() {
      this.appInitSwapRate();

      const {
        ftCodehash,
        ftGenesis,
        ftSymbol,
        ftSensibleId,
        ftBalance,
        ftDecimalNum,
      } = this.toAccount;
      if (!this.from || !this.recevice) {
        this.$toast.fail({
          message: `${this.$t("unBlank")}`,
          forbidClick: true,
          duration: 3000,
        });
        this.swapLoading = false;
        return;
      }
      if (+this.from > +ftBalance && this.token1 !== "BSV") {
        this.$toast.fail({
          message: `${this.$t("mcAmountEnough")}`,
          forbidClick: true,
          duration: 3000,
        });
        this.swapLoading = false;
        return;
      }
      if (+this.from > +this.fromAccount.bsv && this.token1 === "BSV") {
        this.$toast.fail({
          message: `${this.$t("bsvAmountEnough")}`,
          forbidClick: true,
          duration: 3000,
        });
        this.swapLoading = false;
        return;
      }
      const swap2symbol = ftSymbol.toLowerCase();
      const symbol = `bsv-mc`;
      const bsvOption = {
        amount:
          this.token1 === "BSV"
            ? +this.bsvToSatoshis
            : this.recevice * Math.pow(10, 8),
        codehash: "",
        genesisId: "",
      };
      const ftOption = {
        amount:
          this.token2 === "BSV"
            ? this.from * 10 ** +this.toAccount.ftDecimalNum
            : this.recevice * Math.pow(10, +this.toAccount.ftDecimalNum),
        codehash: ftCodehash,
        genesisId: ftGenesis,
        sensibleId: ftSensibleId,
      };
      this.swapLoading = true;
      const op_code = this.token1 === "BSV" ? 3 : 4;
      const swapRate = this.initRateDate;
      if (this.token1 === "BSV" && this.initRateDate.op == 3) {
        let swapFee =
          bsvOption.amount +
          this.initRateDate.txFee +
          (bsvOption.amount * this.initRateDate.swapFeeRate) / 10000;
        if (swapFee > +this.fromAccount.bsv * 10 ** 8) {
          this.$toast.fail({
            message: this.$t("changeFeeNotEnought"),
            forbidClick: true,
            duration: 3000,
          });
          this.swapLoading = false;
          return;
        }
      } else {
        let swapFee = this.initRateDate.txFee;
        if (swapFee > +this.fromAccount.bsv * 10 ** 8) {
          this.$toast.fail({
            message: this.$t("changeFeeNotEnought"),
            forbidClick: true,
            duration: 3000,
          });
          this.swapLoading = false;
          return;
        }
      }
      window.appMetaIdJsV2.swapft(
        this.appAccessToken,
        symbol,
        op_code,
        JSON.stringify(bsvOption),
        JSON.stringify(ftOption),
        JSON.stringify(swapRate),
        "appToDetails"
      );
    },
    appToDetails(res) {
      let result;
      try {
        result = JSON.parse(res);
      } catch (err) {
        result = res;
        this.swapLoading = false;
      }

      if (result.code == 0) {
        this.swapLoading = false;
        this.swapTxid = result.swapTxid;
        this.toDetailPage();
      } else {
        this.swapLoading = false;
        this.$toast.fail({
          message:
            result.indexOf("swapft failed") !== -1
              ? `${this.$t("bsvEnough")}`
              : result?.message
              ? `${result.message}`
              : "token兑换失败",
          forbidClick: true,
          duration: 3000,
        });
      }
    },
    toDetailPage() {
      this.$router.push({
        name: "Detail",
        params: {
          from: {
            fromName: this.token1 === "BSV" ? "BSV" : this.toAccount.ftSymbol,
            fromBalance: this.from,
            fromIcon: this.token1 === "BSV" ? "BSVIcon" : this.toAccount.ftIcon,
          },
          to: {
            toName: this.token2 === "BSV" ? "BSV" : this.toAccount.ftSymbol,
            toBalance: this.recevice,
            toIcon: this.token2 === "BSV" ? "BSVIcon" : this.toAccount.ftIcon,
          },
          tx: this.swapTxid,
        },
      });
    },
  
    //切换Token
    async toggleCoin() {
      this.from = null;
      this.recevice = 0;
      this.priceFloat = 0;

      switch (this.token1) {
        case "BSV":
          this.token2 = this.token1;
          this.token1 = this.selectFtToken;
          this.appMetaIdJsV2
            ? this.appInitSwapRate()
            : await this.initSwapRate();

          break;
        case this.selectFtToken:
          this.token1 = this.token2;
          this.token2 = this.selectFtToken;
          this.appMetaIdJsV2
            ? this.appInitSwapRate()
            : await this.initSwapRate();

          break;
        default:
          break;
      }
    },

    openDialog(tokenType) {
      this.dialogShow = true;
      if (tokenType === "BSV") {
        this.bsvDisable = true;
      } else {
        this.ftDisable = true;
      }
    },
    //获取FT Token列表
    getFtList() {
      this.$store.state.metaidjs.getFTList({
        accessToken: this.accessToken,
        data: {},
        callback: (res) => {
          if (res.code == 200 && res.data.length) {
            this.ftList = res.data;
            let ftItem = this.ftList.filter((item) => {
              return item && item.ftName === "MetaCoin";
            });
            this.$store.dispatch("asyncSetToAccount", ...ftItem);
          }
        },
      });
    },
    //app获取FT Token列表
    appGetFtList() {
      window.appMetaIdJsV2.ftList(this.appAccessToken, "appGetFT");
    },
    appGetFT(res) {
      const result = JSON.parse(res);
      this.ftList = result;
      let ftItem = this.ftList.filter((item) => {
        return item && item.ftName === "MetaCoin";
      });
      this.$store.dispatch("asyncSetToAccount", ...ftItem);
    },

    //获取BSV余额
    getBsvAccount() {
      this.$store.state.metaidjs.getBalance({
        accessToken: this.accessToken,
        data: {},
        callback: (res) => {
          if (res.code == 200) {
            this.$store.dispatch("asyncSetFromAccount", res.data);
          }
        },
      });
    },
    //app获取BSV余额
    appGetBsvAccount() {
      window.appMetaIdJsV2.getBalance(this.appAccessToken, "getBsv");
    },
    getBsv(res) {
      const result = JSON.parse(res);

      if (result.code == 200) {
        let data = {};
        data.satoshi = 0;
        data.bsv = +result.data;
        this.$store.dispatch("asyncSetFromAccount", data);
      }
    },
  },

  created() {
    this.appMetaIdJsV2 = window.appMetaIdJsV2;
    if (window.appMetaIdJsV2) {
      window.getBsv = this.getBsv;
      window.toggleFt = this.toggleFt;
      window.appInitSwapreqswapargs = this.appInitSwapreqswapargs;
      window.appToDetails = this.appToDetails;
      window.appGetFT = this.appGetFT;
    }
  },
  mounted() {},
};
</script>

<style lang="scss" scoped>
.selectionWrap {
  .top {
    padding-top: 0.857143rem /* 12/14 */;
    display: flex;
    justify-content: space-between;
    align-items: center;
    color: #909399;
    font-size: 0.928571rem /* 13/14 */;
  }
  .foot {
    margin-top: 0.714286rem /* 10/14 */;
    padding: 0 0.857143rem /* 12/14 */;
    background-color: #f7f9fc;
    border-radius: 0.714286rem /* 10/14 */;
    display: flex;
    justify-content: space-between;
    align-items: center;

    .van-cell {
      width: 28.571429rem /* 400/14 */;
      height: 4.714286rem /* 66/14 */;
      background-color: #f7f9fc;
      padding: 0.142857rem /* 2/14 */ /* 21/14 */ 0;
      font-size: 1.142857rem /* 16/14 */;
      // line-height: 4.714286rem /* 66/14 */;
      // text-align: center;
      display: flex;
      align-items: center;
    }
    ::v-deep .van-cell.van-field {
      .van-cell__value.van-cell__value--alone.van-field__value {
        .van-field__body {
          .van-field__control {
            height: 3.928571rem /* 55/14 */;
            display: flex;
            align-items: center;
            // max-width: 100%;
          }
        }
      }
    }
    .ftList {
      height: 2.714286rem /* 38/14 */;
      display: flex;
      align-items: center;
      background-color: #fff;
      border-radius: 0.357143rem /* 5/14 */;
      padding: 0 0.857143rem /* 12/14 */;
      i {
        color: #ffc051;
        font-size: 1.571429rem !important;
      }
      img {
        width: 1.571429rem /* 22/14 */;
        height: 1.571429rem /* 22/14 */;
        border-radius: 50%;
      }
      span {
        font-size: 0.928571rem /* 13/14 */;
        padding-left: 0.357143rem /* 5/14 */;
      }
    }
  }
  .PayDetailWrap {
    margin-top: 2.142857rem /* 30/14 */;
    .top {
      padding-top: 0.857143rem /* 12/14 */;
      display: flex;
      justify-content: space-between;
      align-items: center;
      color: #909399;
      font-size: 0.928571rem /* 13/14 */;
    }
    ::v-deep .van-button.van-button--info.van-button--normal {
      background-color: #01bafb;
      border: 1px solid #01bafb;
      border-radius: 0.714286rem /* 10/14 */;
      width: 100%;
      margin-top: 0.785714rem /* 11/14 */;
      height: 4.285714rem /* 60/14 */;
      .van-button__content {
        .van-button__text {
          padding: 1.357143rem /* 19/14 */ 0;
          font-size: 1.142857rem /* 16/14 */;

          letter-spacing: 0.214286rem; /* 3/14 */
        }
      }
    }
  }
}
@media screen and (max-width: 500px) {
  .selectionWrap {
    .foot {
      .van-cell {
        width: 14.285714rem /* 200/14 */;
      }
    }
  }
}
</style>
