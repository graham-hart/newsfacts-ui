<template>
  <div id="app">
    <h2>{{ category.name }}</h2>
    <div v-if="votes.length > 0" id="avgVotes">
      <p class="catText text">
        <br />
        <b>{{ avgVotes }}</b>
        out of <b>{{ votes.length }}</b> votes
      </p>
    </div>
    <div v-else id="avgVotes">
      <p class="catText text">
        <br />
        This category doesn't have any votes. <br />Be the first person to vote!
      </p>
    </div>

    <span :key="setUserVote()">
      <input
        type="range"
        id="vote"
        v-model="vote"
        step="1"
        :min="category.range_min"
        :max="category.range_max"
        @change="voteChanged()"
        :disabled="!isAuthenticated"
      />
      <h2 id="votedisplay" v-if="isAuthenticated">{{ vote }}</h2>
      <span v-else class="text"><p>Please log in to vote!</p></span>
      <button v-if="isAuthenticated" @click="submitVote()" id="voteSubmit">
        Vote
      </button>
    </span>
    <BarChart
      :key="this.category.name"
      :category="this.category"
      :chartData="chartData"
    />
  </div>
</template>
<script>
import api from "@/scripts/api.js";
import BarChart from "@/components/BarChart.vue";
export default {
  name: "Rating",
  components: { BarChart },
  data() {
    return { vote: null, voteSet: false, changed: false };
  },
  mounted() {
    this.setUserVote();
  },
  props: {
    category: Object,
    site: Object,
  },
  methods: {
    voteChanged() {
      this.voteSet = true;
    },
    setUserVote() {
      if (!this.voteSet) {
        let v = this.getUserVote;
        if (v != null) {
          this.voteSet = true;
          this.vote = v;
        } else if (this.vote == null) {
          // this.voteSet = true;
          this.vote = 0;
        }
        return this.vote;
      }
    },
    submitVote() {
      let v = this.votes.filter(
        (v) =>
          v.newssite_id == this.site.newssite_id &&
          v.category_id == this.category.category_id &&
          v.voter_email == this.$auth.user.email
      )[0];
      try {
        api.patch(this.$store, `vote?vote_id=eq.${v.vote_id}`, {
          score: this.vote,
        });
      } catch (e) {
        api.post(this.$store, `vote`, {
          score: this.vote,
          newssite_id: this.site.newssite_id,
          category_id: this.category.category_id,
          voter_email: this.$auth.user.email,
        });
      }
      this.$store.commit("refreshData");
      this.$forceUpdate();
    },
  },
  computed: {
    isAuthenticated() {
      return this.$auth.isAuthenticated;
    },
    votes() {
      return this.$store.state.vote.filter(
        (v) =>
          v.newssite_id == this.site.newssite_id &&
          v.category_id == this.category.category_id
      );
    },
    user() {
      return this.$store.state.user;
    },
    getUserVote() {
      let v;
      try {
        v = this.votes.filter(
          (v) =>
            (v.category_id == this.category.category_id &&
              v.newssite_id == this.site.newssite_id &&
              v.voter_email == this.$auth.user.email) ||
            ""
        )[0];
      } catch {
        v = null;
      }
      if (!v) {
        v = null;
      } else {
        v = v.score;
      }
      return v;
    },
    avgVotes() {
      let sum = 0;
      let num = 0;
      for (let v of this.votes) {
        sum++;
        num += v.score;
      }
      return Math.round((num / sum) * 100) / 100;
    },
    votesByScore() {
      let scFr = {};
      this.votes.forEach((v) => {
        scFr[v.score] = 0;
      });
      this.votes.forEach((v) => {
        scFr[v.score] += 1;
      });
      for (let k of this.labels) {
        if (!Object.keys(scFr).includes(k.toString())) {
          scFr[k] = 0;
        }
      }
      const keysArr = Object.keys(scFr).sort(
        (a, b) => parseInt(a) - parseInt(b)
      );
      let vals = [];
      keysArr.forEach((k) => {
        vals.push(scFr[k]);
      });
      return vals;
    },
    chartData() {
      return {
        labels: this.labels,
        datasets: [
          {
            label: "Number of votes",
            data: this.votesByScore,
            backgroundColor: "#5332a8bb",
            hoverBackgroundColor: "#5332a8",
          },
        ],
      };
    },
    labels() {
      let l = [];
      for (let i = this.category.range_min; i <= this.category.range_max; i++) {
        l.push(i);
      }
      return l;
    },
  },
};
</script>
<style scoped>
#app {
  margin: 20px auto;
  background-color: white;
  flex-grow: 1;
  width: 600px;
  padding: 20px;
  border-radius: 10px;
  border: 8px solid #e1e1e1;
}
#vote {
  --margin-h: 20px;
  --margin-v: 1%;
  width: calc(100% - calc(var(--margin-h) * 2));
  margin: var(--margin-v) var(--margin-h);
}
#votedisplay {
  text-align: center;
}
h2 {
  font-weight: 800;
  text-align: center;
}
#voteSubmit {
  margin: auto !important;
  display: block;
  padding: 5px;
  color: white;
  background-color: #5327a8;
  box-shadow: 1px 3px 2px #000000aa;
}
.catText {
  font-weight: 400;
}
.text {
  font-size: 20px;
  text-align: center;
}
</style>