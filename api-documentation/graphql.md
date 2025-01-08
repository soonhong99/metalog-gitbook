# GraphQL 스키마 (보완 필요)

각 API는 독립적으로 운영되면서도, 일관된 스키마 구조를 통해 클라이언트 개발자들에게 직관적인 인터페이스를 제공

특히 실시간 기능이 중요한 Post와 Room API는 WebSocket을 통한 Subscription을 제공하여 즉각적인 업데이트가 가능하도록 설계

### 1. Core Types

모든 API에서 공통으로 사용되는 기본 타입들

```graphql
# 공통 인터페이스
interface Node {
  id: ID!
  createdAt: DateTime!
  updatedAt: DateTime
}

# 페이지네이션 인풋
input PaginationInput {
  first: Int
  after: String
  last: Int
  before: String
}

# 기본 메타데이터
type MetaData {
  totalCount: Int!
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
}
```

### 2. Post API (PostQL)

```graphql
type Post implements Node {
  id: ID!
  createdAt: DateTime!
  updatedAt: DateTime
  title: String!
  content: String!
  author: User!
  metaAssets: [MetaAsset!]
  tags: [String!]!
  visibility: VisibilityType!
  reactions: ReactionConnection!
  comments: CommentConnection!
}

enum VisibilityType {
  PUBLIC
  PRIVATE
  FRIENDS_ONLY
}

type ReactionConnection {
  edges: [ReactionEdge!]!
  pageInfo: PageInfo!
  totalCount: Int!
}

type CommentConnection {
  edges: [CommentEdge!]!
  pageInfo: PageInfo!
  totalCount: Int!
}

input CreatePostInput {
  title: String!
  content: String!
  metaAssetIds: [ID!]
  tags: [String!]!
  visibility: VisibilityType!
}

type PostQuery {
  post(id: ID!): Post
  posts(
    filter: PostFilterInput
    pagination: PaginationInput
  ): PostConnection!
  trendingPosts: [Post!]!
  userPosts(
    userId: ID!
    pagination: PaginationInput
  ): PostConnection!
}

type PostMutation {
  createPost(input: CreatePostInput!): Post!
  updatePost(id: ID!, input: UpdatePostInput!): Post!
  deletePost(id: ID!): Boolean!
  addReaction(postId: ID!, type: ReactionType!): Reaction!
  removeReaction(postId: ID!, reactionId: ID!): Boolean!
}

type PostSubscription {
  postCreated: Post!
  postUpdated(id: ID!): Post!
  newReaction(postId: ID!): Reaction!
}
```

### 3. Room API (RoomQL)

```graphql
type Room implements Node {
  id: ID!
  createdAt: DateTime!
  updatedAt: DateTime
  owner: User!
  name: String!
  description: String
  metaAssets: [MetaAsset!]!
  customization: RoomCustomization!
  visitors: VisitorConnection!
  visibility: VisibilityType!
}

type RoomCustomization {
  theme: String!
  layout: String!
  backgroundColor: String
  decorations: [MetaAsset!]!
}

type VisitorConnection {
  edges: [VisitorEdge!]!
  pageInfo: PageInfo!
  totalCount: Int!
}

input CreateRoomInput {
  name: String!
  description: String
  metaAssetIds: [ID!]!
  customization: RoomCustomizationInput!
  visibility: VisibilityType!
}

type RoomQuery {
  room(id: ID!): Room
  rooms(
    filter: RoomFilterInput
    pagination: PaginationInput
  ): RoomConnection!
  randomRooms(count: Int!): [Room!]!
  popularRooms: [Room!]!
}

type RoomMutation {
  createRoom(input: CreateRoomInput!): Room!
  updateRoom(id: ID!, input: UpdateRoomInput!): Room!
  customizeRoom(id: ID!, customization: RoomCustomizationInput!): Room!
  addMetaAsset(roomId: ID!, assetId: ID!): Room!
}

type RoomSubscription {
  roomUpdated(id: ID!): Room!
  newVisitor(roomId: ID!): Visit!
}
```

### 4. Recommendation API (RecommendQL)

```graphql
type Recommendation {
  id: ID!
  type: RecommendationType!
  target: RecommendationTarget!
  score: Float!
  reason: String
}

enum RecommendationType {
  POST
  ROOM
  USER
  META_ASSET
}

union RecommendationTarget = Post | Room | User | MetaAsset

type RecommendationQuery {
  recommendedPosts(
    userId: ID!
    pagination: PaginationInput
  ): PostConnection!
  recommendedRooms(
    userId: ID!
    pagination: PaginationInput
  ): RoomConnection!
  recommendedUsers(
    userId: ID!
    pagination: PaginationInput
  ): UserConnection!
  trendingTags(limit: Int): [Tag!]!
}

type RecommendationSubscription {
  newRecommendation(userId: ID!): Recommendation!
}
```

### 5. Notification API (NotificationQL)

```graphql
type Notification implements Node {
  id: ID!
  createdAt: DateTime!
  updatedAt: DateTime
  type: NotificationType!
  recipient: User!
  actor: User
  target: NotificationTarget
  read: Boolean!
}

enum NotificationType {
  POST_REACTION
  POST_COMMENT
  ROOM_VISIT
  FOLLOW
  SYSTEM_NOTICE
}

union NotificationTarget = Post | Room | Comment | User

type NotificationQuery {
  notifications(
    filter: NotificationFilterInput
    pagination: PaginationInput
  ): NotificationConnection!
  unreadCount: Int!
}

type NotificationMutation {
  markAsRead(id: ID!): Notification!
  markAllAsRead: Boolean!
  updateNotificationSettings(input: NotificationSettingsInput!): NotificationSettings!
}

type NotificationSubscription {
  notificationReceived(userId: ID!): Notification!
}
```

### 6. 스키마 설계 원칙

1. **Interface 활용**
   * `Node` 인터페이스를 통한 일관된 ID 관리
   * 공통 필드 표준화
2. **Connection 패턴**
   * 페이지네이션 일관성
   * 엣지 기반 관계 표현
   * 메타데이터 포함
3. **Type 세분화**
   * 명확한 enum 타입 정의
   * union 타입을 통한 다형성 지원
   * 입력값 검증을 위한 별도 Input 타입
4. **실시간 처리**
   * Subscription 타입을 통한 실시간 업데이트
   * WebSocket 연동
   * 이벤트 기반 알림

