<template>
  <div class="rating-content">
    <!-- 新建评分表单 -->
    <div v-if="showNewRatingForm" class="new-rating-form">
      <h2 class="form-title">新建评分</h2>
      <form @submit.prevent="submitNewRating">
        <div class="form-group">
          <label for="rating-goal">评分对象</label>
          <input type="text" id="rating-goal" v-model="newRating.goal" required placeholder="输入评分对象名称">
        </div>

        <div class="form-group">
          <label for="rating-tag">分类标签</label>
          <select id="rating-tag" v-model="newRating.tag" required>
            <option value="" disabled>请选择分类</option>
            <option v-for="tag in tagOptions" :key="tag" :value="tag">{{ tag }}</option>
          </select>
        </div>

        <div class="form-group">
          <label for="rating-intro">简介描述</label>
          <textarea
              id="rating-intro"
              v-model="newRating.intro"
              required
              placeholder="描述评分对象的基本信息"
              rows="4"
          ></textarea>
        </div>

        <div class="form-group">
          <label for="rating-image">封面图片</label>
          <input type="file" id="rating-image" @change="handleImageUpload">
          <p class="help-text">*支持jpg, png格式，建议尺寸600×400px</p>
        </div>        <div class="form-group">
          <label>您的评分 <span class="optional-text">(可选)</span></label>
          <div class="rating-input">
            <div class="star-rating">
              <span
                  v-for="n in 5"
                  :key="n"
                  class="star"
                  :class="{ 'active': n <= newRating.score }"
                  @click="newRating.score = n"
              >★</span>
            </div>
            <span class="rating-value">{{ newRating.score ? `${newRating.score}分` : '未评分' }}</span>
            <button type="button" class="reset-rating-btn" @click="newRating.score = 0">清除评分</button>
          </div>
          <p class="help-text">*可以先创建评分对象，稍后再进行评分</p>
        </div><div class="form-group">
          <label for="rating-comment">评价内容 <span class="optional-text">(可选)</span></label>
          <textarea
              id="rating-comment"
              v-model="newRating.comment"
              placeholder="分享您的体验和评价（可选，创建评分后会自动添加为评论）"
              rows="5"
          ></textarea>
          <p class="help-text">*如果填写，将在创建评分后自动添加为您的评论</p>
        </div>

        <div class="form-actions">
          <button type="button" class="cancel-btn" @click="cancelNewRating">取消</button>
          <button type="submit" class="submit-btn">发布评分</button>
        </div>
      </form>
    </div>

    <!-- 评分详情视图 -->
    <div v-else-if="currentViewMode === 'detail' && currentRating" class="rating-detail-view">
      <button class="back-btn" @click="backToList">返回列表</button>

      <div class="detail-header">
        <div class="detail-img-container">
          <img :src="getImageUrl(currentRating.image)" :alt="currentRating.goal" class="detail-img">
          <div class="detail-tag">{{ currentRating.tag }}</div>
        </div>

        <div class="detail-title-section">
          <h2 class="detail-title">{{ currentRating.goal }}</h2>
          <div class="detail-score-container">
            <div class="detail-score">{{ currentRating.score ? currentRating.score.toFixed(1) : '0.0' }}</div>
            <div class="detail-stars">
              <div class="stars-container detail-stars-bg">
                <div class="stars-filled" :style="{width: ((currentRating.score || 0)/5)*100 + '%'}"></div>
              </div>
              <div class="detail-participants">{{ currentRating.num }}人参与</div>
            </div>
          </div>
        </div>
      </div>

      <div class="detail-body">
        <div class="detail-intro">
          <h3 class="section-subtitle">介绍</h3>
          <p>{{ currentRating.intro }}</p>
        </div>        <div class="rating-comments-section">
          <h3 class="section-subtitle">评论 ({{ currentRating.comments ? currentRating.comments.length : 0 }})</h3>

          <!-- 我的评分 -->
          <div class="my-rating" v-if="!hasRated">
            <h4 class="my-rating-title">我的评分</h4>
            <div class="rating-input">
              <div class="star-rating">
                <span
                    v-for="n in 5"
                    :key="n"
                    class="star"
                    :class="{ 'active': n <= userRating }"
                    @click="setUserRating(n)"
                >★</span>
              </div>
              <span class="rating-value">{{ userRating }}</span>
            </div>
            <textarea
                v-model="userComment"
                class="comment-input"
                placeholder="分享你的评价..."
            ></textarea>
            <button class="submit-btn" @click="submitRating">提交评价</button>
          </div>          <!-- 评论列表 -->
          <div class="comments-list">
            <div
                v-for="comment in (currentRating.comments || [])"
                :key="`${comment.sid}-${comment.uid}`"
                class="comment-item"
            >
              <div class="comment-header">
                <div class="user-avatar">
                  <img :src="comment.userAvatar" :alt="comment.userName">
                </div>
                <div class="comment-user-info">
                  <div class="comment-user-name">{{ comment.userName }}</div>
                  <div class="comment-time">{{ formatTime(comment.time) }}</div>
                </div>
                <div class="comment-score">
                  <div class="comment-stars">
                    <span v-for="n in 5" :key="n" class="comment-star" :class="{ 'filled': n <= comment.score }">★</span>
                  </div>
                  <span class="comment-score-value">{{ comment.score }}</span>
                </div>
              </div>
              <div class="comment-content">{{ comment.comment }}</div>
            </div>
          </div>
        </div>
      </div>
    </div>

    <!-- 筛选和排序 -->
    <div v-else>
      <div class="filter-sort-container">
        <div class="sort-options">
          <span class="sort-label">排序:</span>
          <button
              class="sort-btn"
              :class="{ active: sortBy === 'newest' }"
              @click="setSortBy('newest')"
          >
            最新
          </button>
          <button
              class="sort-btn"
              :class="{ active: sortBy === 'hottest' }"
              @click="setSortBy('hottest')"
          >
            热门
          </button>
          <button
              class="sort-btn"
              :class="{ active: sortBy === 'highest' }"
              @click="setSortBy('highest')"
          >
            最高分
          </button>
        </div>
      </div>

      <!-- 评分列表 -->
      <div class="rating-list" v-if="filteredRatings.length > 0">
        <div
            v-for="rating in filteredRatings"
            :key="rating.sid"
            class="rating-card"
            @click="viewRatingDetails(rating.sid)"
        >
          <div class="rating-img-container">
            <img :src="getImageUrl(rating.image)" :alt="rating.goal" class="rating-img">
            <div class="rating-tag">{{ rating.tag }}</div>
          </div>
          <div class="rating-info">
            <h3 class="rating-title">{{ rating.goal }}</h3>
            <div class="rating-stats">
              <div class="rating-score">
                <span class="score">{{ rating.score ? rating.score.toFixed(1) : '0.0' }}</span>
                <div class="stars-container">
                  <div class="stars-filled" :style="{width: ((rating.score || 0)/5)*100 + '%'}"></div>
                </div>
              </div>
              <div class="rating-participants">{{ rating.num }}人参与</div>
            </div>
            <p class="rating-intro">{{ rating.intro }}</p>
          </div>
        </div>
      </div>

      <!-- 无结果提示 -->
      <div class="no-results" v-else>
        <p>没有找到相关评分内容</p>
      </div>
    </div>
  </div>
</template>

<script>
import { ratingsApi, userApi } from '@/api';
import { baseURL } from '@/api/index';
import { getImageUrl } from '@/utils/imageUtils';

export default {
  name: 'RatingContent',
  props: {
    currentTag: {
      type: String,
      default: null
    },
    searchQuery: {
      type: String,
      default: ''
    },
    showNewPostForm: {
      type: Boolean,
      default: false
    },
    // 新增的props，接收从父组件传来的指定要查看的评分ID
    selectedRatingId: {
      type: Number,
      default: null
    }
  },
  data() {
    return {
      sortBy: 'newest',
      currentViewMode: 'list', // 'list', 'detail'
      currentRatingId: null,
      userRating: 0,
      userComment: '',
      hasRated: false,
      showNewRatingForm: false,
      isLoading: true,
      error: null,
      newRating: {
        goal: '',
        tag: '',
        intro: '',
        image: '/images/default.jpg',
        score: 0,
        comment: ''      },
      ratings: [],
      tagOptions: [],
      userAvatarCache: {}, // 用户头像缓存
      userAvatarPromiseCache: {} // 用户头像Promise缓存，防止并发请求
    }
  },
  computed: {
    filteredRatings() {
      let result = [...this.ratings];

      // 标签过滤
      if (this.currentTag) {
        result = result.filter(rating => rating.tag === this.currentTag);
      }

      // 搜索过滤
      if (this.searchQuery) {
        const query = this.searchQuery.toLowerCase();
        result = result.filter(rating =>
            rating.goal.toLowerCase().includes(query) ||
            rating.tag.toLowerCase().includes(query) ||
            rating.intro.toLowerCase().includes(query)
        );
      }

      // 排序
      switch (this.sortBy) {
        case 'newest':
          // 按时间排序，较新的评分在前
          result.sort((a, b) => b.time - a.time);
          break;
        case 'hottest':
          // 按参与人数排序
          result.sort((a, b) => b.num - a.num);
          break;
        case 'highest':
          // 按分数排序
          result.sort((a, b) => b.score - a.score);
          break;
      }

      return result;
    },    currentRating() {
      const rating = this.ratings.find(r => r.sid === this.currentRatingId);
      if (rating && !rating.comments) {
        rating.comments = [];
      }
      return rating || null;
    }
  },
  watch: {
    showNewPostForm(newVal) {
      if (newVal) {
        this.showNewRatingForm = true;
        this.currentViewMode = 'list';
      }
    },
    currentTag() {
      // 切换标签时回到列表视图
      this.currentViewMode = 'list';
      this.showNewRatingForm = false;
    },
    searchQuery() {
      // 搜索时回到列表视图
      this.currentViewMode = 'list';
      this.showNewRatingForm = false;
    },
    // 监听来自父组件的selectedRatingId变化
    selectedRatingId(newVal) {
      if (newVal) {
        this.viewRatingDetails(newVal);
      }
    },
    // 当ratings数据发生变化时，向父组件发送更新事件
    ratings: {
      deep: true,
      handler(newVal) {
        this.$emit('ratings-updated', newVal);
      }
    }
  },
  created() {
    // 获取评分标签
    this.fetchRatingTags();
    
    // 获取评分列表
    this.fetchRatings();

    // 如果初始selectedRatingId存在，则自动显示该评分详情
    if (this.selectedRatingId) {
      this.viewRatingDetails(this.selectedRatingId);
    }
  },
  methods: {
    // 获取评分标签
    fetchRatingTags() {
      ratingsApi.getRatingTags()
        .then(response => {
          if (response.data && response.data.code === 1) {
            this.tagOptions = response.data.data || [];
          } else {
            console.error('获取评分标签失败:', response.data.msg);
          }
        })
        .catch(error => {
          console.error('获取评分标签出错:', error);
        });
    },    // 获取评分列表
    fetchRatings() {
      this.isLoading = true;
      ratingsApi.getRatings()
        .then(response => {
          if (response.data && response.data.code === 1) {
            // 确保每个评分对象都有comments数组
            const ratingsData = (response.data.data || []).map(rating => {
              if (!rating.comments) {
                rating.comments = [];
              }
              return rating;
            });
            this.ratings = ratingsData;
            // 发送更新事件通知父组件
            this.$emit('ratings-updated', this.ratings);
          } else {
            console.error('获取评分列表失败:', response.data?.msg);
            this.error = response.data?.msg || '获取评分列表失败';
          }
        })
        .catch(error => {
          console.error('获取评分列表出错:', error);
          this.error = '网络错误，请稍后再试';
        })
        .finally(() => {
          this.isLoading = false;
        });
    },    // 获取评分详情
    fetchRatingDetail(id) {
      this.isLoading = true;
      ratingsApi.getRatingDetail(id)
        .then(response => {
          if (response.data && response.data.code === 1) {
            // 确保评分对象有comments数组
            const ratingData = response.data.data;
            if (!ratingData.comments) {
              ratingData.comments = [];
            }
            
            // 查找评分是否已经存在于列表中
            const index = this.ratings.findIndex(r => r.sid === id);
            if (index !== -1) {
              // 更新已有评分
              this.ratings.splice(index, 1, ratingData);
            } else {
              // 添加新评分到列表
              this.ratings.push(ratingData);
            }
            this.currentRatingId = id;
            this.currentViewMode = 'detail';
            
            // 检查用户是否已经评分
            this.checkUserRating();
            
            // 获取该评分的评论
            this.fetchRatingComments(id);
          } else {
            console.error('获取评分详情失败:', response.data?.msg);
            alert('获取评分详情失败: ' + (response.data?.msg || '未知错误'));
          }
        })
        .catch(error => {
          console.error('获取评分详情出错:', error);
          alert('网络错误，请稍后再试');
        })
        .finally(() => {
          this.isLoading = false;
        });
    },    // 获取评分评论
    async fetchRatingComments(id) {
      try {
        const response = await ratingsApi.getRatingComments(id);
        if (response.data && response.data.code === 1) {
          // 找到对应的评分并更新其评论
          const rating = this.ratings.find(r => r.sid === id);
          if (rating) {
            const comments = response.data.data || [];
            
            // 为每个评论获取用户名和头像
            if (comments.length > 0) {
              const enrichedComments = await this.enrichCommentsWithUserInfo(comments);
              rating.comments = enrichedComments;
            } else {
              rating.comments = comments;
            }
          }
        } else {
          console.error('获取评分评论失败:', response.data?.msg);
        }
      } catch (error) {
        console.error('获取评分评论出错:', error);
      }
    },    // 为评论添加用户信息（用户名和头像）
    async enrichCommentsWithUserInfo(comments) {
      const promises = comments.map(async comment => {
        try {
          // 获取用户名
          const userResponse = await userApi.getUsername(comment.uid);
          if (userResponse.data && userResponse.data.code === 1) {
            comment.userName = userResponse.data.data;
          } else {
            comment.userName = `用户${comment.uid}`;
          }
          
          // 获取用户头像
          try {
            comment.userAvatar = await this.fetchUserAvatar(comment.uid);
          } catch (avatarError) {
            console.error(`获取用户${comment.uid}头像失败:`, avatarError);
            comment.userAvatar = `https://via.placeholder.com/48?text=U${comment.uid}`;
          }
          
          return comment;
        } catch (error) {
          console.error(`获取用户${comment.uid}信息失败:`, error);
          comment.userName = `用户${comment.uid}`;
          comment.userAvatar = `https://via.placeholder.com/48?text=U${comment.uid}`;
          return comment;
        }
      });
      
      return await Promise.all(promises);
    },

    setSortBy(sortType) {
      this.sortBy = sortType;
    },
    
    viewRatingDetails(sid) {
      this.showNewRatingForm = false;
      this.fetchRatingDetail(sid);
    },
    
    backToList() {
      this.currentViewMode = 'list';
      this.currentRatingId = null;
    },
    
    setUserRating(rating) {
      this.userRating = rating;
    },
    
    // 检查用户是否已经评分
    checkUserRating() {
      // 在实际应用中，应该从后端获取用户是否已经评分
      // 这里简单模拟
      this.hasRated = false;
      this.userRating = 0;
      this.userComment = '';
    },    async submitRating() {
      if (this.userRating === 0) {
        alert('请先给出您的评分');
        return;
      }

      const ratingData = {
        sid: this.currentRatingId,
        score: this.userRating,
        comment: this.userComment,
        time: Date.now() // 添加时间戳
      };

      this.isSubmitting = true;
      
      try {
        const response = await ratingsApi.commentRating(this.currentRatingId, ratingData);
        if (response.data && response.data.code === 1) {
          this.hasRated = true;
          
          // 重新获取评分详情和评论（包含用户信息）
          await this.fetchRatingDetail(this.currentRatingId);
          
          // 清空评论
          this.userComment = '';
          
          // 通知用户
          alert('评价提交成功！');
        } else {
          console.error('提交评分失败:', response.data.msg);
          alert('提交评分失败: ' + (response.data.msg || '未知错误'));
        }
      } catch (error) {
        console.error('提交评分出错:', error);
        alert('网络错误，请稍后再试');
      } finally {
        this.isSubmitting = false;
      }
    },
    
    formatTime(timestamp) {
      const date = new Date(timestamp);
      return `${date.getFullYear()}-${String(date.getMonth() + 1).padStart(2, '0')}-${String(date.getDate()).padStart(2, '0')} ${String(date.getHours()).padStart(2, '0')}:${String(date.getMinutes()).padStart(2, '0')}`;
    },
      handleImageUpload(event) {
      const file = event.target.files[0];
      if (file) {
        // 文件大小验证
        if (file.size > 2 * 1024 * 1024) {
          alert('图片大小不能超过2MB');
          return;
        }

        // 文件类型验证
        if (!file.type.match(/^image\/(jpeg|jpg|png)$/)) {
          alert('只支持JPG、JPEG、PNG格式的图片');
          return;
        }

        // 创建FormData对象并添加文件
        const formData = new FormData();
        formData.append('file', file);        // 调用API上传图片
        ratingsApi.uploadImage(formData)
          .then(response => {
            if (response.data) {
              // 直接保存服务器返回的相对路径，不拼接host
              this.newRating.image = response.data;
              alert('图片上传成功');
            } else {
              console.error('上传图片失败:', response.data?.msg);
              alert('上传图片失败: ' + (response.data?.msg || '未知错误'));
            }
          })
          .catch(error => {
            console.error('上传图片出错:', error);
            alert('网络错误，请稍后再试');
          });
      }
    },      submitNewRating() {
      // 检查是否已上传图片
      if (!this.newRating.image || this.newRating.image === '/images/default.jpg') {
        alert('请先上传评分图片');
        return;
      }

      // 使用后端Score实体的字段名
      const ratingData = {
        tag: this.newRating.tag,
        goal: this.newRating.goal,
        intro: this.newRating.intro,
        image: this.newRating.image,
        score: this.newRating.score || 0, // 允许分数为0，表示还未评分
        num: 0, // 初始参与人数为0
        status: 0 // 新创建的评分状态为0
      };

      this.isSubmitting = true;
        ratingsApi.createRating(ratingData)
        .then(async response => {
          if (response.data && response.data.code === 1) {
            // 从API响应中获取新创建的评分对象
            const newRating = response.data.data;            // 如果有评价内容，创建评论（无论评分多少）
            if (this.newRating.comment && this.newRating.comment.trim() && newRating?.sid) {
              try {
                console.log('准备创建评论，sid:', newRating.sid);
                const commentData = {
                  sid: newRating.sid,
                  score: this.newRating.score || 0, // 允许评分为0
                  comment: this.newRating.comment.trim(),
                  time: Date.now()
                };
                
                console.log('评论数据:', commentData);
                console.log('调用 add-comment API...');
                
                const commentResponse = await ratingsApi.commentRating(newRating.sid, commentData);
                console.log('评论API响应:', commentResponse);
                
                if (commentResponse.data && commentResponse.data.code === 1) {
                  console.log('评论创建成功');
                } else {
                  console.error('评论创建失败:', commentResponse.data?.msg);
                }
              } catch (commentError) {
                console.error('创建评论出错:', commentError);
                // 即使评论创建失败，也不影响评分的创建成功
              }
            } else {
              console.log('跳过评论创建，原因:', {
                hasComment: !!this.newRating.comment,
                commentTrimmed: this.newRating.comment?.trim(),
                hasSid: !!newRating?.sid,
                sid: newRating?.sid
              });
            }
            
            // 重新获取评分列表以显示新创建的评分和评论
            await this.fetchRatings();
            
            // 重置表单
            this.resetNewRatingForm();
            
            // 重新获取评分列表（确保包含新创建的评论）
            this.fetchRatings();
            
            // 隐藏表单
            this.showNewRatingForm = false;
            this.$emit('hide-new-post-form');
            
            // 通知用户
            alert('评分创建成功！');
          } else {
            console.error('创建评分失败:', response.data.msg);
            alert('创建评分失败: ' + (response.data.msg || '未知错误'));
          }
        })
        .catch(error => {
          console.error('创建评分出错:', error);
          alert('网络错误，请稍后再试');
        })
        .finally(() => {
          this.isSubmitting = false;
        });
    },
    
    cancelNewRating() {
      this.resetNewRatingForm();
      this.showNewRatingForm = false;
      this.$emit('hide-new-post-form');
    },
      resetNewRatingForm() {
      this.newRating = {
        goal: '',
        tag: '',
        intro: '',
        image: '/images/default.jpg',
        score: 0,
        comment: ''
      };
    },
    
    // 获取完整的图片URL
    getImageUrl(imagePath) {
      return getImageUrl(imagePath);
    },

    // 头像获取相关方法
    async fetchUserAvatar(uid) {
      // 检查缓存
      if (this.userAvatarCache[uid]) {
        return this.userAvatarCache[uid];
      }

      // 检查是否已有pending的请求
      if (this.userAvatarPromiseCache[uid]) {
        return this.userAvatarPromiseCache[uid];
      }

      // 创建新的请求Promise
      const promise = this.fetchUserAvatarFromAPI(uid);
      this.userAvatarPromiseCache[uid] = promise;

      try {
        const avatarUrl = await promise;
        this.userAvatarCache[uid] = avatarUrl;
        return avatarUrl;
      } catch (error) {
        console.error('获取用户头像失败:', error);
        const fallbackUrl = `https://via.placeholder.com/48?text=U${uid}`;
        this.userAvatarCache[uid] = fallbackUrl;
        return fallbackUrl;
      } finally {
        // 清除Promise缓存
        delete this.userAvatarPromiseCache[uid];
      }
    },    async fetchUserAvatarFromAPI(uid) {
      try {
        const response = await userApi.getUserData(uid);
        if (response.data && response.data.code === 1 && response.data.data) {
          const userData = response.data.data;
          // 同时处理image_path和image字段
          const imagePath = userData.image_path || userData.image;
          if (imagePath) {
            // 如果头像路径是相对路径，拼接baseURL
            if (imagePath.startsWith('/') || imagePath.startsWith('image/')) {
              const cleanBaseURL = baseURL.endsWith('/') ? baseURL.slice(0, -1) : baseURL;
              return `${cleanBaseURL}/${imagePath.startsWith('/') ? imagePath.substring(1) : imagePath}`;
            }
            // 如果是完整URL，直接返回
            return imagePath;
          }
        }
        // 如果没有头像数据，返回默认头像
        return `https://via.placeholder.com/48?text=U${uid}`;
      } catch (error) {
        console.error('从API获取用户头像失败:', error);
        // 发生错误时也返回默认头像
        return `https://via.placeholder.com/48?text=U${uid}`;
      }
    },

    getUserAvatar(userObj) {
      if (!userObj || !userObj.uid) {
        return 'https://via.placeholder.com/48?text=U';
      }
      
      const cached = this.userAvatarCache[userObj.uid];
      if (cached) {
        return cached;
      }
      
      // 异步获取头像，但不阻塞渲染
      this.fetchUserAvatar(userObj.uid);
      
      // 返回占位符，等待头像加载完成后会自动更新
      return `https://via.placeholder.com/48?text=U${userObj.uid}`;
    },
  }
}
</script>

<style scoped>
.rating-content {
  background-color: #fff9f7;
  border-radius: 15px;
  padding: 25px;
  box-shadow: 0 4px 6px rgba(139, 0, 0, 0.08);
  height: 100%;
  overflow-y: auto;
  max-height: calc(100vh - 50px);
}

/* ===== 评分列表样式 ===== */
.rating-list {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(320px, 1fr));
  gap: 24px;
  margin-top: 20px;
}

.rating-card {
  background-color: white;
  border-radius: 12px;
  overflow: hidden;
  box-shadow: 0 3px 10px rgba(176, 58, 46, 0.08);
  transition: transform 0.3s ease, box-shadow 0.3s ease;
  cursor: pointer;
  height: 100%;
  display: flex;
  flex-direction: column;
}

.rating-card:hover {
  transform: translateY(-5px);
  box-shadow: 0 8px 18px rgba(176, 58, 46, 0.15);
}

.rating-img-container {
  position: relative;
  height: 180px;
  overflow: hidden;
}

.rating-img {
  width: 100%;
  height: 100%;
  object-fit: cover;
  transition: transform 0.5s ease;
}

.rating-card:hover .rating-img {
  transform: scale(1.05);
}

.rating-tag {
  position: absolute;
  top: 12px;
  right: 12px;
  background-color: rgba(176, 58, 46, 0.9);
  color: white;
  padding: 4px 12px;
  border-radius: 20px;
  font-size: 0.75rem;
  font-weight: 600;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  backdrop-filter: blur(2px);
}

.rating-info {
  padding: 18px;
  flex-grow: 1;
  display: flex;
  flex-direction: column;
}

.rating-title {
  margin: 0 0 12px;
  font-size: 1.3rem;
  color: #4b2e2e;
  font-weight: 600;
  line-height: 1.3;
}

.rating-stats {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 12px;
}

.rating-score {
  display: flex;
  align-items: center;
  gap: 10px;
}

.score {
  color: #B03A2E;
  font-size: 1.6rem;
  font-weight: 700;
}

.stars-container {
  position: relative;
  width: 80px;
  height: 16px;
  background: url('data:image/svg+xml;utf8,<svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 16 16"><path fill="%23D8D8D8" d="M8 1.3l2.5 5 5.5.8-4 3.9.9 5.4-4.9-2.6L3.1 16l.9-5.4-4-3.9 5.5-.8z"/></svg>') repeat-x;
  background-size: 16px 16px;
}

.stars-filled {
  position: absolute;
  top: 0;
  left: 0;
  height: 100%;
  background: url('data:image/svg+xml;utf8,<svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 16 16"><path fill="%23F6B352" d="M8 1.3l2.5 5 5.5.8-4 3.9.9 5.4-4.9-2.6L3.1 16l.9-5.4-4-3.9 5.5-.8z"/></svg>') repeat-x;
  background-size: 16px 16px;
}

.rating-participants {
  color: #6c757d;
  font-size: 0.85rem;
  font-weight: 500;
  background-color: #f8f9fa;
  padding: 3px 8px;
  border-radius: 12px;
}

.rating-intro {
  color: #6c757d;
  font-size: 0.95rem;  line-height: 1.6;
  margin: 0;
  display: -webkit-box;
  -webkit-line-clamp: 3;
  line-clamp: 3;
  -webkit-box-orient: vertical;
  overflow: hidden;
  flex-grow: 1;
}

/* ===== 筛选和排序 ===== */
.filter-sort-container {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 20px;
  flex-wrap: wrap;
  gap: 15px;
  background-color: white;
  padding: 15px;
  border-radius: 12px;
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.03);
}

.sort-options {
  display: flex;
  gap: 10px;
  align-items: center;
}

.sort-label {
  color: #5C2E2E;
  font-size: 0.9rem;
  font-weight: 500;
}

.sort-btn {
  padding: 6px 14px;
  background: transparent;
  border: 1px solid #F1C6B5;
  border-radius: 20px;
  color: #5C2E2E;
  font-size: 0.85rem;
  cursor: pointer;
  transition: all 0.2s ease;
}

.sort-btn:hover {
  background-color: #FDEDEA;
}

.sort-btn.active {
  background-color: #B03A2E;
  color: white;
  border-color: #B03A2E;
  font-weight: 500;
  box-shadow: 0 2px 5px rgba(176, 58, 46, 0.2);
}

/* ===== 详情视图样式 ===== */
.rating-detail-view {
  background-color: white;
  border-radius: 15px;
  box-shadow: 0 3px 10px rgba(0, 0, 0, 0.05);
  overflow: hidden;
}

.back-btn {
  margin: 15px;
  padding: 8px 16px;
  background-color: #f8f9fa;
  border: none;
  border-radius: 8px;
  color: #5C2E2E;
  cursor: pointer;
  font-size: 0.9rem;
  display: flex;
  align-items: center;
  gap: 5px;
  transition: background-color 0.2s ease;
}

.back-btn:hover {
  background-color: #FDEDEA;
}

.back-btn::before {
  content: "←";
  font-size: 1.1rem;
}

.detail-header {
  position: relative;
}

.detail-img-container {
  position: relative;
  height: 250px;
  overflow: hidden;
}

.detail-img {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.detail-tag {
  position: absolute;
  top: 15px;
  right: 15px;
  background-color: rgba(176, 58, 46, 0.9);
  color: white;
  padding: 5px 14px;
  border-radius: 20px;
  font-size: 0.85rem;
  font-weight: 600;
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.2);
}

.detail-title-section {
  position: absolute;
  bottom: 0;
  left: 0;
  right: 0;
  background: linear-gradient(to top, rgba(0, 0, 0, 0.8), transparent);
  padding: 30px 20px 20px;
  color: white;
}

.detail-title {
  font-size: 2rem;
  margin: 0 0 15px;
  font-weight: 700;
  text-shadow: 0 1px 3px rgba(0, 0, 0, 0.3);
}

.detail-score-container {
  display: flex;
  align-items: center;
  gap: 15px;
}

.detail-score {
  font-size: 2rem;
  font-weight: bold;
  color: #F6B352;
  text-shadow: 0 1px 2px rgba(0, 0, 0, 0.3);
}

.detail-stars {
  display: flex;
  flex-direction: column;
  gap: 5px;
}

.detail-stars-bg {
  width: 120px;
  height: 24px;
  background-size: 24px 24px;
}

.detail-stars .stars-filled {
  background-size: 24px 24px;
}

.detail-participants {
  font-size: 0.9rem;
  color: rgba(255, 255, 255, 0.9);
}

.detail-body {
  padding: 25px;
}

.section-subtitle {
  color: #5C2E2E;
  font-size: 1.3rem;
  margin: 0 0 15px;
  padding-bottom: 10px;
  border-bottom: 2px solid #F1C6B5;
  font-weight: 600;
}

.detail-intro {
  margin-bottom: 35px;
}

.detail-intro p {
  color: #4b4b4b;
  line-height: 1.7;
  font-size: 1rem;
}

/* ===== 评论区域 ===== */
.rating-comments-section {
  margin-top: 20px;
}

.my-rating {
  background-color: #FDEDEA;
  padding: 22px;
  border-radius: 12px;
  margin-bottom: 25px;
  box-shadow: 0 2px 8px rgba(176, 58, 46, 0.1);
}

.my-rating-title {
  margin: 0 0 15px;
  color: #B03A2E;
  font-size: 1.1rem;
  font-weight: 600;
}

.rating-input {
  display: flex;
  align-items: center;
  gap: 15px;
  margin-bottom: 15px;
}

.star-rating {
  display: flex;
  gap: 5px;
}

.star {
  font-size: 1.8rem;
  color: #D8D8D8;
  cursor: pointer;
  transition: color 0.2s ease;
  user-select: none;
}

.star:hover, .star.active {
  color: #F6B352;
}

.rating-value {
  font-size: 1.3rem;
  font-weight: bold;
  color: #B03A2E;
}

.comment-input {
  width: 100%;
  padding: 14px;
  border: 1px solid #ddd;
  border-radius: 10px;
  resize: vertical;
  min-height: 100px;
  margin-bottom: 15px;
  font-family: inherit;
  font-size: 0.95rem;
  transition: border-color 0.2s ease, box-shadow 0.2s ease;
}

.comment-input:focus {
  outline: none;
  border-color: #EBA293;
  box-shadow: 0 0 0 2px rgba(235, 162, 147, 0.2);
}

.submit-btn {
  padding: 10px 24px;
  background-color: #B03A2E;
  color: white;
  border: none;
  border-radius: 10px;
  font-size: 1rem;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.3s ease;
  box-shadow: 0 2px 5px rgba(176, 58, 46, 0.2);
}

.submit-btn:hover {
  background-color: #912D23;
  transform: translateY(-2px);
  box-shadow: 0 4px 8px rgba(176, 58, 46, 0.25);
}

.submit-btn:active {
  transform: translateY(0);
}

/* 评论列表 */
.comments-list {
  display: flex;
  flex-direction: column;
  gap: 18px;
}

.comment-item {
  background-color: white;
  padding: 18px;
  border-radius: 12px;
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.05);
  border: 1px solid #f0f0f0;
  transition: transform 0.2s ease, box-shadow 0.2s ease;
}

.comment-item:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.08);
}

.comment-header {
  display: flex;
  align-items: center;
  margin-bottom: 12px;
}

.user-avatar {
  width: 48px;
  height: 48px;
  border-radius: 50%;
  overflow: hidden;
  margin-right: 12px;
  border: 2px solid #F1C6B5;
}

.user-avatar img {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.comment-user-info {
  flex: 1;
}

.comment-user-name {
  font-weight: 600;
  color: #4b2e2e;
  margin-bottom: 4px;
  font-size: 1.05rem;
}

.comment-time {
  font-size: 0.8rem;
  color: #6c757d;
}

.comment-score {
  display: flex;
  flex-direction: column;
  align-items: flex-end;
}

.comment-stars {
  display: flex;
  gap: 2px;
}

.comment-star {
  font-size: 1rem;
  color: #D8D8D8;
}

.comment-star.filled {
  color: #F6B352;
}

.comment-score-value {
  font-size: 1rem;
  font-weight: 600;
  color: #B03A2E;
  margin-top: 2px;
}

.comment-content {
  color: #4b4b4b;
  line-height: 1.6;
  font-size: 1rem;
}

/* ===== 新建评分表单 ===== */
.new-rating-form {
  background-color: white;
  border-radius: 15px;
  padding: 25px;
  box-shadow: 0 3px 15px rgba(0, 0, 0, 0.05);
}

.form-title {
  color: #B03A2E;
  font-size: 1.5rem;
  margin: 0 0 25px;
  text-align: center;
  font-weight: 600;
}

.form-group {
  margin-bottom: 20px;
}

.form-group label {
  display: block;
  color: #4b2e2e;
  font-weight: 500;
  margin-bottom: 8px;
  font-size: 1rem;
}

.optional-text {
  color: #6c757d;
  font-weight: 400;
  font-size: 0.85rem;
}

.form-group input[type="text"],
.form-group select,
.form-group textarea {
  width: 100%;
  padding: 12px 15px;
  border: 1px solid #ddd;
  border-radius: 10px;
  font-size: 0.95rem;
  color: #333;
  background-color: #fff;
  transition: border-color 0.2s ease, box-shadow 0.2s ease;
}

.form-group input[type="text"]:focus,
.form-group select:focus,
.form-group textarea:focus {
  outline: none;
  border-color: #EBA293;
  box-shadow: 0 0 0 2px rgba(235, 162, 147, 0.2);
}

.form-group input[type="file"] {
  padding: 10px 0;
}

.help-text {
  font-size: 0.8rem;
  color: #6c757d;
  margin-top: 5px;
}

.form-actions {
  display: flex;
  justify-content: flex-end;
  gap: 15px;
  margin-top: 30px;
}

.cancel-btn {
  padding: 10px 20px;
  background-color: #f8f9fa;
  border: 1px solid #ddd;
  border-radius: 10px;
  color: #4b4b4b;
  font-size: 1rem;
  cursor: pointer;
  transition: all 0.2s ease;
}

.cancel-btn:hover {
  background-color: #e9ecef;
}

/* ===== 无结果提示 ===== */
.no-results {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 50px 0;
  text-align: center;
  background-color: white;
  border-radius: 15px;
  margin-top: 20px;
}

.empty-image {
  width: 180px;
  margin-bottom: 25px;
  opacity: 0.7;
}

.no-results p {
  color: #6c757d;
  margin-bottom: 25px;
  font-size: 1.1rem;
}

.new-rating-btn {
  padding: 12px 24px;
  background-color: #B03A2E;
  color: white;
  border: none;
  border-radius: 10px;
  font-size: 1rem;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.3s ease;
  box-shadow: 0 2px 5px rgba(176, 58, 46, 0.2);
}

.new-rating-btn:hover {
  background-color: #912D23;
  transform: translateY(-2px);
  box-shadow: 0 4px 8px rgba(176, 58, 46, 0.25);
}

/* ===== 响应式设计 ===== */
@media (max-width: 992px) {
  .rating-list {
    grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
  }

  .detail-title {
    font-size: 1.6rem;
  }
}

@media (max-width: 768px) {
  .rating-content {
    padding: 15px;
  }

  .filter-sort-container {
    flex-direction: column;
    align-items: stretch;
    padding: 12px;
  }

  .sort-options {
    justify-content: space-between;
  }

  .rating-list {
    grid-template-columns: 1fr;
    gap: 18px;
  }

  .detail-img-container {
    height: 200px;
  }

  .detail-title {
    font-size: 1.4rem;
  }

  .form-actions {
    flex-direction: column;
  }

  .cancel-btn, .submit-btn {
    width: 100%;
  }
}

@media (max-width: 480px) {
  .detail-score-container {
    flex-direction: column;
    align-items: flex-start;
    gap: 8px;
  }

  .comment-header {
    flex-wrap: wrap;
  }

  .comment-score {
    margin-top: 10px;
    width: 100%;
    align-items: flex-start;
  }

  .user-avatar {
    width: 40px;
    height: 40px;
  }

  .star {
    font-size: 1.5rem;
  }
}
</style>