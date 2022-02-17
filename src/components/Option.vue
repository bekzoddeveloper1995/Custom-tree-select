<script>
  import { UNCHECKED, INDETERMINATE, CHECKED } from '../constants'
  import { onLeftClick } from '../utils'
  import Tip from './Tip'
  import ArrowIcon from './icons/Arrow'

  let arrowPlaceholder, checkMark, minusMark

  const Option = {
    name: 'vue-treeselect--option',
    inject: [ 'instance' ],

    props: {     
      node: {
        type: Object,
        required: true,
      },
    },

    computed: {
      shouldExpand() {
        const { instance, node } = this

        return node.isBranch && instance.shouldExpand(node)
      },

      shouldShow() {
        const { instance, node } = this

        return instance.shouldShowOptionInMenu(node)
      },
    },

    methods: {
      renderOption() {
        const { instance, node} = this
        const optionClass = {
          'vue-treeselect__option ': true,
          'font-acrom-bold':(node.isBranch && !node.ancestors.length ),
           'font-acrom-medium':(node.isBranch && node.ancestors.length ),
           'font-acrom-regular ':(!node.isBranch && node.ancestors.length ),
          'vue-treeselect__option--disabled': node.isDisabled,
          'vue-treeselect__option--selected': instance.isSelected(node),
          'vue-treeselect__option--highlight': node.isHighlighted,
          'vue-treeselect__option--matched': instance.localSearch.active && node.isMatched,
          'vue-treeselect__option--hide': !this.shouldShow,
        }
       
        return (
          <div class={optionClass} onMouseenter={this.handleMouseEnterOption} data-id={node.id}>           
            {this.renderLabelContainer([
              !node.isBranch && node.ancestors.length ? <svg onMousedown={this.handleMouseDownOnIcon} style="margin-right:10px" width="20" height="20" viewBox="0 0 20 20" fill="none" xmlns="http://www.w3.org/2000/svg">
                 <path d="M10 0.25C4.624 0.25 0.25 4.624 0.25 10C0.25 15.376 4.624 19.75 10 19.75C15.376 19.75 19.75 15.376 19.75 10C19.75 4.624 15.376 0.25 10 0.25ZM10 1.75C14.5645 1.75 18.25 5.4355 18.25 10C18.25 14.5645 14.5645 18.25 10 18.25C5.4355 18.25 1.75 14.5645 1.75 10C1.75 5.4355 5.4355 1.75 10 1.75ZM9.25 5.5V7H10.75V5.5H9.25ZM9.25 8.5V14.5H10.75V8.5H9.25Z" fill="#949494"/>
              </svg> : null,            
               this.renderLabel(),
               this.renderCheckboxContainer([
               this.renderCheckbox()
              ]),
            ])}
             {this.renderArrow()}
          </div>
        )
      },

      renderSubOptionsList() {
        if (!this.shouldExpand) return null

        return (
          <div class="vue-treeselect__list dssd">
            {this.renderSubOptions()}
            {this.renderNoChildrenTip()}
            {this.renderLoadingChildrenTip()}
            {this.renderLoadingChildrenErrorTip()}
          </div>
        )
      },

      renderArrow() {
        const { instance, node } = this

        if (instance.shouldFlattenOptions && this.shouldShow) return null

        if (node.isBranch) {
          const transitionProps = {
            props: {
              name: 'vue-treeselect__option-arrow--prepare',
              appear: true,
            },
          }
          const arrowClass = {
            'vue-treeselect__option-arrow': true,
            'vue-treeselect__option-arrow--rotated': this.shouldExpand,
          }

          return (
            <div class="vue-treeselect__option-arrow-container" onMousedown={this.handleMouseDownOnArrow}>
              <transition {...transitionProps}>
                <ArrowIcon class={arrowClass} />
              </transition>
            </div>
          )
        }

        // For leaf nodes, we render a placeholder to keep its label aligned to
        // branch nodes. Unless there is no branch nodes at all (a normal
        // non-tree select).
        if (/*node.isLeaf && */instance.hasBranchNodes) {
          if (!arrowPlaceholder) arrowPlaceholder = (
            <div class="vue-treeselect__option-arrow-placeholder">&nbsp;</div>
          )

          return arrowPlaceholder
        }

        return null
      },

      renderLabelContainer(children) {
        return (
          <div class="vue-treeselect__label-container" onMousedown={this.handleMouseDownOnLabelContainer}>
            {children}
          </div>
        )
      },

      renderCheckboxContainer(children) {
        const { instance, node } = this

        if (instance.single) return null
        if (instance.disableBranchNodes && node.isBranch) return null
        if (node.isBranch) return null
        return (
          <div class="vue-treeselect__checkbox-container">
            {children}
          </div>
        )
      },

      renderCheckbox() {
        const { instance, node } = this
        const checkedState = instance.forest.checkedStateMap[node.id]
        const checkboxClass = {
          'vue-treeselect__checkbox': true,
          'vue-treeselect__checkbox--checked': checkedState === CHECKED,
          'vue-treeselect__checkbox--indeterminate': checkedState === INDETERMINATE,
          'vue-treeselect__checkbox--unchecked': checkedState === UNCHECKED,
          'vue-treeselect__checkbox--disabled': node.isDisabled,
        }

        if (!checkMark) checkMark = (
          <span class="vue-treeselect__check-mark" />
        )
        if (!minusMark) minusMark = (
          <span class="vue-treeselect__minus-mark" />
        )

        return (
         <span class={checkboxClass}>
            {checkMark}
            {minusMark}
          </span> 
        )
      },

      renderLabel() {
        const { instance, node } = this
        const shouldShowCount = (
          node.isBranch && (instance.localSearch.active
            ? instance.showCountOnSearchComputed
            : instance.showCount
          )
        )
        const count = shouldShowCount
          ? instance.localSearch.active
            ? instance.localSearch.countMap[node.id][instance.showCountOf]
            : node.count[instance.showCountOf]
          : NaN
        const labelClassName = 'vue-treeselect__label'
        const countClassName = 'vue-treeselect__count'
        const customLabelRenderer = instance.$scopedSlots['option-label']

        if (customLabelRenderer) return customLabelRenderer({
          node,
          shouldShowCount,
          count,
          labelClassName,
          countClassName,
        })

        return (
          <label class={labelClassName}>
            {node.label}
            {shouldShowCount && (
              <span class={countClassName}>({count})</span>
            )}
          </label>
        )
      },

      renderSubOptions() {
        const { node } = this

        if (!node.childrenStates.isLoaded) return null

        return node.children.map(childNode => (
          <Option node={childNode} key={childNode.id} />
        ))
      },

      renderNoChildrenTip() {
        const { instance, node } = this

        if (!node.childrenStates.isLoaded || node.children.length) return null

        return (
          <Tip type="no-children" icon="warning">{ instance.noChildrenText }</Tip>
        )
      },

      renderLoadingChildrenTip() {
        const { instance, node } = this

        if (!node.childrenStates.isLoading) return null

        return (
          <Tip type="loading" icon="loader">{ instance.loadingText }</Tip>
        )
      },

      renderLoadingChildrenErrorTip() {
        const { instance, node } = this

        if (!node.childrenStates.loadingError) return null

        return (
          <Tip type="error" icon="error">
            { node.childrenStates.loadingError }
            <a class="vue-treeselect__retry" title={instance.retryTitle} onMousedown={this.handleMouseDownOnRetry}>
              { instance.retryText }
            </a>
          </Tip>
        )
      },

      handleMouseEnterOption(evt) {
        const { instance, node } = this

        // Equivalent to `self` modifier.
        // istanbul ignore next
        if (evt.target !== evt.currentTarget) return

        instance.setCurrentHighlightedOption(node, false)
      },

      handleMouseDownOnIcon: onLeftClick(function handleMouseDownOnOptionArrow() {
        const { instance, node } = this
        if (typeof instance.func === 'function') {
            instance.func.call(this)
        }
      }),

      handleMouseDownOnArrow: onLeftClick(function handleMouseDownOnOptionArrow() {
        const { instance, node } = this

        instance.toggleExpanded(node)
      }),

      handleMouseDownOnLabelContainer: onLeftClick(function handleMouseDownOnLabelContainer() {
        const { instance, node } = this

        if (node.isBranch && instance.disableBranchNodes) {
          instance.toggleExpanded(node)
        } else {
          instance.select(node)
        }
      }),

      handleMouseDownOnRetry: onLeftClick(function handleMouseDownOnRetry() {
        const { instance, node } = this

        instance.loadChildrenOptions(node)
      }),
    },

    render() {
      const { node } = this
      const indentLevel = this.instance.shouldFlattenOptions ? 0 : node.level
      const listItemClass = {
        'vue-treeselect__list-item': true,
        [`vue-treeselect__indent-level-${indentLevel}`]: true,
      }
      const transitionProps = {
        props: {
          name: 'vue-treeselect__list--transition',
        },
      }

      return (
        <div class={listItemClass}>
          {this.renderOption()}
          {node.isBranch && (
            <transition {...transitionProps}>
              {this.renderSubOptionsList()}
            </transition>
          )}
        </div>
      )
    },
  }

  
  export default Option
</script>
