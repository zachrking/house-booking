<template>
    <div class="bb-comment">
        <comment-header :recommend="recommend" :has-rating="hasRating" :email-data="getUser"/>
        <comment-box :has-rating="hasRating" :email="email"/>
        <div class="bb-loading" v-if="isLoading"></div>
        <div class="bb-comment-list" v-if="!isLoading">
            <comment-item v-for="(comment, index) in comments" :key="comment.id" :comment="comment" :on-delete-item="() => onDeletedItem(index)" />
            <div class="bb-comment-list-empty text-center" v-if="!comments.length">
                <p>{{ __('Become the first to comment') }}</p>
            </div>
        </div>
        <confirm-dialog v-if="confirmDialogData"
                        :title="confirmDialogData.title"
                        :message="confirmDialogData.message"
                        :on-done="confirmDialogData.onDone"
                        :on-close="() => this.confirmDialogData = null"
        />

        <login-form :show.sync="showLoginForm" :on-close="closeModalLogin" :onLoginWithGuard="onLoginWithGuard" :loginWithGuards="loginWithGuards"></login-form>

        <div class="bb-comment-footer d-flex justify-content-center" v-if="reactive.attrs">
            <button v-if="reactive.attrs.last_page > reactive.attrs.current_page" @click="loadMoreComments" :class="'btn btn-secondary' + (isLoadMore ? ' button-loading' : '')">{{ __('Load More') }}</button>
        </div>

        <div class="bb-comment-loading" v-if="isSoftLoading">
            <div class="bb-loading mini"></div>
        </div>

        <comment-footer v-if="copyright" :text="copyright" />
    </div>
</template>

<script>
import CommentBox from "./partials/CommentBox";
import CommentItem from "./partials/CommentItem";
import CommentHeader from "./partials/CommentHeader";
import ConfirmDialog from "./partials/ConfirmDialog";
import CommentFooter from "./partials/CommentFooter";
import LoginForm from "./partials/LoginForm";
import Http from '../service/http';
import Ls from '../service/Ls';
window.Pusher = require('pusher-js');
window.http = Http;

export default {
    data: () => {
        return {
            isLoading: true,
            isLoadMore: false,
            error: false,
            comments: [],
            recommend: {},
            email: '',
            reactive: {
                userData: null,
                attrs: null,
                sort: 'newest',
                rating: {},
            },
            confirmDialogData: null,
            showLoginForm: false,
            isSoftLoading: false,
            loginWithGuards: [],
        };
    },
    components: {
        CommentBox,
        CommentItem,
        CommentHeader,
        ConfirmDialog,
        LoginForm,
        CommentFooter,

    },
    props: {
        reference: {
            type: String,
            required: true
        },
        url: {
            type: String,
            required: true
        },
        postUrl: {
            type: String,
            required: true
        },
        userUrl: {
            type: String,
            required: true,
        },
        deleteUrl: {
            type: String,
            required: true,
        },
        likeUrl: {
            type: String,
            required: true,
        },
        loginUrl: {
            type: String,
            required: true,
        },
        logoutUrl: {
            type: String,
            required: true,
        },
        registerUrl: {
            type: String,
            required: true,
        },
        changeAvatarUrl: {
            type: String,
            required: true,
        },
        recommendUrl: {
            type: String,
            required: true,
        },
        loggedUser: {
            type: Object,
        },
        checkCurrentUserApi: {
            type: String,
            required: true,
        },
        captcha: {
            type: String,
            default: ''
        },
        copyright: {
            type: String,
        }
    },
    computed: {
        hasRating() {
            return !!Object.keys(this.reactive.rating);
        }
    },
    methods: {
        getUser(email) {
           this.email = email;
           console.log(this.email);
        },
        async onLoginWithGuard(user) {
            this.setSoftLoading(true);
            try {
                const res = await Http.post(this.checkCurrentUserApi + '?guard=' + user.key);
                this.setSoftLoading(false);
                this.showLoginForm = false;
                if (res) {
                    if (!res.data.error) {
                        Ls.set('auth.token', res.data.data.token);
                        await this.getCurrentUserData();
                    } else {
                        alert(res.data?.message || 'Error! An error occurred. Please try again later');
                    }
                }
            } catch (e) {
                this.setSoftLoading(false)
            }
        },
        async getCurrentUserData() {
            if (Ls.get('auth.token')) {
                this.setSoftLoading(true);
                const res = await Http.post(this.userUrl);
                this.reactive.userData = res.data.data;
                this.setSoftLoading(false)
            } else {
                this.reactive.userData = null;
            }

            this.loadComments();
        },
        apiLoadComments(params, cb) {
            console.log(this.url);
            Http.get(this.url, {
                params: {
                    reference: this.reference,
                    sort: this.reactive.sort,
                    _t: new Date().getTime(),
                    ...params,
                }
            })
                .then(({ data }) => {
                    if (data.error) {
                        this.error = true;
                        return;
                    }

                    this.error = false;
                    cb(data);
                })
        },
        loadComments() {
            this.isLoading = true;
            this.apiLoadComments({

            }, data => {
                this.isLoading = false;
                this.comments = data.data.comments;
                this.reactive.userData = data.data.user;
                this.reactive.attrs = data.data.attrs;
                this.reactive.rating = data.data.rating;
                this.recommend = data.data.recommend;
            })
        },
        listen() {
            var self = this;
            window.Echo.channel('post')
            .listen('.Botble\\Comment\\Events\\NewCommentEvent', (e) => {
                self.reactive.attrs.count_all += 1;
                if(e.comment.parent_id == "0") {
                    self.comments.unshift(e.comment);
                }
            });
        },
        loadMoreComments() {
            if (this.url && this.reactive.attrs) {
                this.isLoadMore = true;
                this.apiLoadComments({
                    page: this.reactive.attrs.current_page + 1
                }, data => {
                    this.isLoadMore = false;
                    this.comments = this.comments.concat(data.data.comments);
                    this.reactive.attrs = {
                        ...data.data.attrs,
                        count_all: this.reactive.attrs.count_all,
                    };
                })
            }
        },
        showConfirmDialog(title, message, onDone) {
            this.confirmDialogData = {
                title,
                message,
                onDone
            }
        },
        onDeletedItem(index) {
            this.comments.splice(index, 1)
        },
        updateCount(adding = true) {
            if (!adding) {
                this.reactive.attrs.count_all -= 1;
            }
        },
        closeModalLogin(isSuccess = false) {
            this.showLoginForm = false;
            if (isSuccess === true) {
                this.getCurrentUserData();
            }
        },
        openLoginForm() {
            this.checkCurrentUser((data = []) => {
                this.loginWithGuards = data;
                this.showLoginForm = true;
            });
        },
        onChangeSort(sort) {
            this.reactive.sort = sort.value;
            this.loadComments();
        },
        async checkCurrentUser(cb) {
            this.setSoftLoading(true);
            try {
                const res = await Http.post(this.checkCurrentUserApi)
                if (res) {
                    this.setSoftLoading(false);
                    if (!res.data.error) {
                        cb(res.data.data);
                    } else {
                        cb();
                    }
                }
            } catch(e) {
                cb();
                this.setSoftLoading(false)
            }
        },
        setSoftLoading(value = true) {
            this.isSoftLoading = value;
        }
    },
    mounted: function () {

        this.listen();
        this.loadComments();
    },
    provide() {
        return {
            getUser: this.getCurrentUserData,
            data: this.reactive,
            loggedUser: this.loggedUser,
            reference: this.reference,
            postUrl: this.postUrl,
            deleteUrl: this.deleteUrl,
            likeUrl: this.likeUrl,
            showConfirm: this.showConfirmDialog,
            updateCount: this.updateCount,
            openLoginForm: this.openLoginForm,
            apiLoadComments: this.apiLoadComments,
            onChangeSort: this.onChangeSort,
            setSoftLoading: this.setSoftLoading,
            changeAvatarUrl: this.changeAvatarUrl,
            registerUrl: this.registerUrl,
            logoutUrl: this.logoutUrl,
            loginUrl: this.loginUrl,
            recommendUrl: this.recommendUrl,
            captcha: this.captcha,
        }
    }
}
</script>
