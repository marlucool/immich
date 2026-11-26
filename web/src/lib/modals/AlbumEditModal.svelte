<script lang="ts">
  import AlbumCover from '#lib/components/album-page/AlbumCover.svelte';
  import { authManager } from '#lib/managers/auth-manager.svelte.ts';
  import { getAllAlbumsQuery, updateAlbumInfoMutation } from '#lib/queries/album.query.ts';
  import { executeMutation, queryClient } from '#lib/queries/client.ts';
  import { AlbumUserRole, type AlbumResponseDto } from '@immich/sdk';
  import { Field, FormModal, Input, Select, Textarea, type SelectOption } from '@immich/ui';
  import { createQuery } from '@tanstack/svelte-query';
  import { mdiRenameOutline } from '@mdi/js';
  import { t } from 'svelte-i18n';

  type NestedAlbum = AlbumResponseDto & { parentId?: string | null };

  type Props = {
    album: AlbumResponseDto;
    onClose: () => void;
  };

  let { album, onClose }: Props = $props();

  const albumsQuery = createQuery(() => getAllAlbumsQuery({ isOwned: true }), () => queryClient);
  const albums = $derived((albumsQuery.data ?? []) as NestedAlbum[]);
  const isOwner = $derived(
    album.albumUsers.some(({ user, role }) => user.id === authManager.user.id && role === AlbumUserRole.Owner),
  );

  let albumName = $state(album.albumName);
  let description = $state(album.description ?? '');
  let parentAlbumId = $state((album as NestedAlbum).parentId ?? '');

  const parentAlbumOptions = $derived.by(() => {
    const excludedIds = new Set([album.id]);
    let changed = true;
    while (changed) {
      changed = false;
      for (const candidate of albums) {
        if (candidate.parentId && excludedIds.has(candidate.parentId) && !excludedIds.has(candidate.id)) {
          excludedIds.add(candidate.id);
          changed = true;
        }
      }
    }

    return [
      { label: $t('no_parent_album'), value: '' },
      ...albums
        .filter((candidate) => !excludedIds.has(candidate.id))
        .map((candidate) => ({ label: candidate.albumName || $t('unnamed_album'), value: candidate.id })),
    ] as SelectOption<string>[];
  });

  const onSubmit = async () => {
    const updateAlbumDto = {
      albumName,
      description: description || null,
      ...(isOwner ? { parentId: parentAlbumId || null } : {}),
    };
    await executeMutation(updateAlbumInfoMutation, [{ id: album.id, updateAlbumDto }], { onSuccess: onClose });
  };
</script>

<FormModal icon={mdiRenameOutline} title={$t('edit_album')} size="medium" {onClose} {onSubmit}>
  <div class="m-4 flex items-center gap-8">
    <AlbumCover {album} class="hidden size-50 shadow-lg sm:flex" />

    <div class="flex grow flex-col gap-4">
      <Field label={$t('name')}>
        <Input bind:value={albumName} />
      </Field>

      <Field label={$t('description')}>
        <Textarea bind:value={description} />
      </Field>

      {#if isOwner}
        <Field label={$t('parent_album')}>
          <Select bind:value={parentAlbumId} options={parentAlbumOptions} />
        </Field>
      {/if}
    </div>
  </div>
</FormModal>
