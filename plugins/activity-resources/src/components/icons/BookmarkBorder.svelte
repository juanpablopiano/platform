<!--
// Copyright © 2023 Hardcore Engineering Inc.
//
// Licensed under the Eclipse Public License, Version 2.0 (the "License");
// you may not use this file except in compliance with the License. You may
// obtain a copy of the License at https://www.eclipse.org/legal/epl-2.0
//
// Unless required by applicable law or agreed to in writing, software
// distributed under the License is distributed on an "AS IS" BASIS,
// WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
//
// See the License for the specific language governing permissions and
// limitations under the License.
-->
<script lang="ts">
  export let size: 'small' | 'medium' | 'large'
  export let fill: string = 'currentColor'
</script>

<svg class="svg-{size}" {fill} viewBox="0 0 20 20" xmlns="http://www.w3.org/2000/svg">
  <path
    d="M16.2,18c-0.1,0-0.2,0-0.3-0.1l-6-4l-6,4c-0.2,0.1-0.4,0.1-0.5,0c-0.2-0.1-0.3-0.3-0.3-0.4V4.2 C3.2,3,4.3,2,5.5,2h8.9c1.3,0,2.3,1,2.3,2.2v13.3c0,0.2-0.1,0.4-0.3,0.4C16.4,18,16.3,18,16.2,18z M10,12.8c0.1,0,0.2,0,0.3,0.1 l5.5,3.6V4.2c0-0.6-0.6-1.2-1.3-1.2H5.5C4.8,3,4.2,3.5,4.2,4.2v12.4l5.5-3.6C9.8,12.9,9.9,12.8,10,12.8z"
  />
</svg>
