<template>
  <div
    class="posts"
    v-if="posts.length"
  >
    <div
      class="post"
      v-for="post in posts"
    >
      <router-link :to="post.path">
        <h3>{{new Date(post.frontmatter.date) | formatDate }} - {{post.title}}</h3>
      </router-link>
    </div>
  </div>
</template>

<script>
export default {
  props: ["page"],
  filters: {
    formatDate: function(v) {
      return v.getFullYear() + "/" + (v.getMonth() + 1) + "/" + v.getDate();
    }
  },
  computed: {
    posts() {
      let currentPage = this.page ? this.page : this.$page.path;
      let posts = this.$site.pages
        .filter(x => {
          return x.path.match(new RegExp(`(${currentPage})(?=.*html)`));
        })
        .sort((a, b) => {
          return new Date(b.frontmatter.date) - new Date(a.frontmatter.date);
        });
      return posts;
    }
  }
};
</script>