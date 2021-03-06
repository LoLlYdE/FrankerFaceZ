<template lang="html">
	<div
		:class="{ maximized: maximized || exclusive, exclusive, faded }"
		class="ffz-dialog ffz-main-menu tw-elevation-3 tw-c-background-alt tw-c-text-base tw-border tw-flex tw-flex-nowrap tw-flex-column"
	>
		<header ref="header" class="tw-c-background-base tw-full-width tw-align-items-center tw-flex tw-flex-nowrap" @dblclick="maybeResize($event)">
			<h3 class="ffz-i-zreknarf ffz-i-pd-1">FrankerFaceZ</h3>
			<div class="tw-flex-grow-1 tw-pd-x-2">
				<div class="tw-search-input">
					<label for="ffz-main-menu.search" class="tw-hide-accessible">{{ t('main-menu.search', 'Search Settings') }}</label>
					<div class="tw-relative">
						<div class="tw-absolute tw-align-items-center tw-c-text-alt-2 tw-flex tw-full-height tw-input__icon tw-justify-content-center tw-left-0 tw-top-0 tw-z-default">
							<figure class="ffz-i-search" />
						</div>
						<input
							id="ffz-main-menu.search"
							v-model="query"
							:placeholder="t('main-menu.search', 'Search Settings')"
							type="search"
							class="tw-block tw-border-radius-medium tw-font-size-6 tw-full-width tw-input tw-pd-l-3 tw-pd-r-1 tw-pd-y-05"
							autocapitalize="off"
							autocorrect="off"
							autocomplete="off"
							spellcheck="false"
						>
					</div>
				</div>
			</div>
			<button v-if="!maximized && !exclusive" class="tw-button-icon tw-mg-x-05" @click="faded = ! faded">
				<span class="tw-button-icon__icon">
					<figure :class="faded ? 'ffz-i-eye-off' : 'ffz-i-eye'" />
				</span>
			</button>
			<button v-if="!exclusive" class="tw-button-icon tw-mg-x-05 tw-tooltip-wrapper" @click="popout">
				<span class="tw-button-icon__icon">
					<figure class="ffz-i-link-ext" />
				</span>
				<div class="tw-tooltip tw-tooltip--down tw-tooltip--align-center">
					{{ t('main-menu.popout', 'Open Settings in a New Window') }}
				</div>
			</button>
			<button v-if="!exclusive" class="tw-button-icon tw-mg-x-05" @click="resize">
				<span class="tw-button-icon__icon">
					<figure :class="{'ffz-i-window-maximize': !maximized, 'ffz-i-window-restore': maximized}" />
				</span>
			</button>
			<button v-if="!exclusive" class="tw-button-icon tw-mg-x-05" @click="close">
				<span class="tw-button-icon__icon">
					<figure class="ffz-i-window-close" />
				</span>
			</button>
		</header>
		<section class="tw-border-t tw-full-height tw-full-width tw-flex tw-flex-nowrap tw-overflow-hidden">
			<nav class="ffz-vertical-nav tw-c-background-alt-2 tw-border-r tw-full-height tw-flex tw-flex-column tw-flex-shrink-0 tw-flex-nowrap">
				<header class="tw-border-b tw-pd-1">
					<profile-selector
						:context="context"
						@navigate="navigate"
					/>
				</header>
				<div class="tw-full-width tw-full-height tw-overflow-hidden tw-flex tw-flex-nowrap tw-relative">
					<simplebar classes="ffz-vertical-nav__items tw-full-width tw-flex-grow-1">
						<menu-tree
							:current-item="currentItem"
							:modal="nav"
							:filter="filter"
							@change-item="changeItem"
							@navigate="navigate"
						/>
					</simplebar>
				</div>
				<footer class="tw-c-text-alt tw-border-t tw-pd-1">
					<div>
						{{ t('main-menu.version', 'Version %{version}', {version: version.toString()}) }}
					</div>
					<div class="tw-c-text-alt-2">
						<a
							v-if="version.commit"
							:href="`https://www.github.com/FrankerFaceZ/FrankerFaceZ/commit/${version.commit}`"
							class="tw-link tw-link--inherit"
							target="_blank"
							rel="noopener noreferrer"
						>
							{{ version.commit.slice(0,7) }}
						</a>
						<span v-else>
							{{ version.build }}
						</span>
					</div>
				</footer>
			</nav>
			<simplebar classes="tw-flex-grow-1">
				<menu-page
					v-if="currentItem"
					ref="page"
					:context="context"
					:item="currentItem"
					:filter="filter"
					@change-item="changeItem"
					@navigate="navigate"
				/>
			</simplebar>
		</section>
	</div>
</template>

<script>

import displace from 'displacejs';

export default {
	data() {
		return this.$vnode.data;
	},

	computed: {
		filter() {
			return this.query.toLowerCase()
		}
	},

	watch: {
		maximized() {
			this.updateDrag();
		}
	},

	created() {
		this.context.context._add_user();
	},

	destroyed() {
		this.context.context._remove_user();
	},

	mounted() {
		this.updateDrag();

		this._on_resize = this.handleResize.bind(this);
		window.addEventListener('resize', this._on_resize);
	},

	beforeDestroy() {
		this.destroyDrag();

		if ( this._on_resize ) {
			window.removeEventListener('resize', this._on_resize);
			this._on_resize = null;
		}
	},

	methods: {
		maybeResize(event) {
			if ( event.target !== this.$refs.header )
				return;

			this.resize(event);
		},

		changeProfile() {
			const new_id = this.$refs.profiles.value,
				new_profile = this.context.profiles[new_id];

			if ( new_profile )
				this.context.currentProfile = new_profile;
		},

		changeItem(item) {
			if ( this.$refs.page && this.$refs.page.onBeforeChange ) {
				if ( this.$refs.page.onBeforeChange(this.currentItem, item) === false )
					return;
			}

			this.currentItem = item;
			let current = item;
			while(current = current.parent) // eslint-disable-line no-cond-assign
				current.expanded = true;
		},

		updateDrag() {
			if ( this.maximized )
				this.destroyDrag();
			else
				this.createDrag();
		},

		destroyDrag() {
			if ( this.displace ) {
				this.displace.destroy();
				this.displace = null;
			}
		},

		createDrag() {
			this.$nextTick(() => {
				if ( ! this.maximized )
					this.displace = displace(this.$el, {
						handle: this.$el.querySelector('header'),
						highlightInputs: true,
						constrain: true
					});
			})
		},

		handleResize() {
			if ( this.displace )
				this.displace.reinit();
		},

		navigate(key) {
			let item = this.nav_keys[key];
			while(item && item.page)
				item = item.parent;

			if ( ! item )
				return;

			this.changeItem(item);
		}
	}
}
</script>