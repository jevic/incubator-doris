<!-- 
Licensed to the Apache Software Foundation (ASF) under one
or more contributor license agreements.  See the NOTICE file
distributed with this work for additional information
regarding copyright ownership.  The ASF licenses this file
to you under the Apache License, Version 2.0 (the
"License"); you may not use this file except in compliance
with the License.  You may obtain a copy of the License at

  http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing,
software distributed under the License is distributed on an
"AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
KIND, either express or implied.  See the License for the
specific language governing permissions and limitations
under the License.
-->

# SHOW FRONTENDS
## Description
This statement is used to view FE nodes
Grammar:
SHOW FRONTENDS;

Explain:
1. name denotes the name of the FE node in bdbje.
2. Join is true to indicate that the node has joined the cluster. But it doesn't mean that it's still in the cluster (it may be out of touch)
3. Alive indicates whether the node survives.
4. Replayed Journal Id represents the maximum metadata log ID that the node has currently replayed.
5. LastHeartbeat is the latest heartbeat.
6. IsHelper indicates whether the node is a helper node in bdbje.
7. ErrMsg is used to display error messages when the heartbeat fails.

## keyword
SHOW, FRONTENDS
